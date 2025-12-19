# Chapter 14: 동시성 제어

## 개요

여러 사용자가 동시에 같은 데이터를 수정할 때 발생하는 충돌을 처리하는 방법을 알아봅니다. 낙관적/비관적 동시성 제어 패턴을 다룹니다.

---

## 동시성 문제란?

### Lost Update 문제

```
시간  사용자 A                    사용자 B
─────────────────────────────────────────────────────────
T1   제품 조회 (가격: $100)
T2                               제품 조회 (가격: $100)
T3   가격을 $110으로 수정
T4   저장 성공
T5                               가격을 $120으로 수정
T6                               저장 성공
─────────────────────────────────────────────────────────
결과: A의 변경($110)이 손실됨, 최종 가격 $120
```

### 해결 방법

```
┌─────────────────────────────────────────────────────────────┐
│                     동시성 제어 방법                          │
├─────────────────────────────┬───────────────────────────────┤
│       낙관적 동시성          │         비관적 동시성          │
│   (Optimistic Concurrency)  │   (Pessimistic Concurrency)   │
├─────────────────────────────┼───────────────────────────────┤
│ 충돌이 드물다고 가정          │ 충돌이 많다고 가정             │
│ 저장 시점에 충돌 검사         │ 읽기 시점에 잠금               │
│ 버전/타임스탬프 비교          │ 데이터베이스 락 사용           │
│ 확장성 좋음                  │ 확장성 낮음                   │
│ EF Core 기본 지원            │ Raw SQL 필요                 │
└─────────────────────────────┴───────────────────────────────┘
```

---

## 14.1 낙관적 동시성(Optimistic Concurrency)

### 간략 설명
대부분의 경우 충돌이 발생하지 않는다고 가정하고, 저장 시점에 충돌을 감지합니다.

### 동시성 토큰 설정

```csharp
// 방법 1: ConcurrencyCheck 어트리뷰트
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;

    [ConcurrencyCheck]
    public decimal Price { get; set; }  // 이 속성의 변경 감지
}

// 방법 2: Fluent API
modelBuilder.Entity<Product>()
    .Property(p => p.Price)
    .IsConcurrencyToken();

// 생성되는 SQL
// UPDATE Products SET Name = @Name, Price = @NewPrice
// WHERE Id = @Id AND Price = @OriginalPrice

// 만약 다른 사용자가 Price를 변경했다면 WHERE 조건 불일치 → 0행 영향
```

---

## 14.2 RowVersion/Timestamp 사용

### 간략 설명
전체 행의 버전을 추적하는 특수 컬럼을 사용하여 모든 변경을 감지합니다.

### SQL Server RowVersion

```csharp
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public decimal Price { get; set; }

    [Timestamp]
    public byte[] RowVersion { get; set; } = null!;
}

// Fluent API
modelBuilder.Entity<Product>()
    .Property(p => p.RowVersion)
    .IsRowVersion();

// 생성되는 테이블
// CREATE TABLE Products (
//     Id int NOT NULL,
//     Name nvarchar(max),
//     Price decimal(18,2),
//     RowVersion rowversion NOT NULL  -- 자동 증가
// );
```

### PostgreSQL xmin 사용

```csharp
// PostgreSQL 시스템 컬럼 활용
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;

    [Timestamp]
    public uint Version { get; set; }  // xmin 시스템 컬럼
}

// Fluent API
modelBuilder.Entity<Product>()
    .UseXminAsConcurrencyToken();
```

### 수동 버전 관리

```csharp
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public decimal Price { get; set; }

    [ConcurrencyCheck]
    public Guid Version { get; set; }
}

// SaveChanges 오버라이드
public override async Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
{
    foreach (var entry in ChangeTracker.Entries<Product>()
        .Where(e => e.State == EntityState.Modified))
    {
        entry.Entity.Version = Guid.NewGuid();
    }

    return await base.SaveChangesAsync(cancellationToken);
}
```

---

## 14.3 동시성 충돌 처리

### DbUpdateConcurrencyException

```csharp
public async Task UpdateProductAsync(int productId, decimal newPrice)
{
    var product = await context.Products.FindAsync(productId);
    if (product == null) return;

    product.Price = newPrice;

    try
    {
        await context.SaveChangesAsync();
    }
    catch (DbUpdateConcurrencyException ex)
    {
        // 충돌 발생!
        var entry = ex.Entries.Single();
        await HandleConcurrencyConflictAsync(entry);
    }
}
```

### 충돌 해결 전략

```csharp
private async Task HandleConcurrencyConflictAsync(EntityEntry entry)
{
    var proposedValues = entry.CurrentValues;
    var databaseValues = await entry.GetDatabaseValuesAsync();

    if (databaseValues == null)
    {
        // 엔티티가 삭제됨
        throw new Exception("데이터가 다른 사용자에 의해 삭제되었습니다.");
    }

    var originalValues = entry.OriginalValues;

    // 전략 1: 클라이언트 우선 (Client Wins)
    // 현재 값으로 덮어쓰기
    entry.OriginalValues.SetValues(databaseValues);
    await context.SaveChangesAsync();

    // 전략 2: 데이터베이스 우선 (Database Wins)
    // 데이터베이스 값으로 복원
    entry.CurrentValues.SetValues(databaseValues);
    entry.OriginalValues.SetValues(databaseValues);

    // 전략 3: 사용자에게 선택권
    // 충돌 정보를 반환하고 사용자 결정 대기
}
```

### 속성별 병합 전략

```csharp
private async Task MergeConflictAsync(EntityEntry entry)
{
    var proposedValues = entry.CurrentValues;
    var databaseValues = await entry.GetDatabaseValuesAsync();
    var originalValues = entry.OriginalValues;

    foreach (var property in proposedValues.Properties)
    {
        var proposedValue = proposedValues[property];
        var databaseValue = databaseValues[property];
        var originalValue = originalValues[property];

        // 규칙: 양쪽 모두 변경했으면 충돌, 한쪽만 변경했으면 해당 값 사용
        if (!Equals(originalValue, proposedValue) && !Equals(originalValue, databaseValue))
        {
            // 양쪽 모두 변경 - 충돌!
            if (!Equals(proposedValue, databaseValue))
            {
                throw new ConflictException(
                    $"{property.Name}: 클라이언트={proposedValue}, DB={databaseValue}");
            }
        }
        else if (!Equals(originalValue, databaseValue))
        {
            // DB만 변경 - DB 값 사용
            proposedValues[property] = databaseValue;
        }
        // else: 클라이언트만 변경 - 현재 값 유지
    }

    entry.OriginalValues.SetValues(databaseValues);
    await context.SaveChangesAsync();
}
```

### 재시도 패턴

```csharp
public async Task<bool> UpdateProductWithRetryAsync(
    int productId,
    Action<Product> updateAction,
    int maxRetries = 3)
{
    for (int attempt = 0; attempt < maxRetries; attempt++)
    {
        try
        {
            var product = await context.Products.FindAsync(productId);
            if (product == null) return false;

            updateAction(product);
            await context.SaveChangesAsync();
            return true;
        }
        catch (DbUpdateConcurrencyException)
        {
            if (attempt == maxRetries - 1)
                throw;

            // 엔티티 다시 로드
            context.ChangeTracker.Clear();
            await Task.Delay(100 * (attempt + 1));  // 백오프
        }
    }

    return false;
}

// 사용
await UpdateProductWithRetryAsync(1, product =>
{
    product.Price = 150;
});
```

---

## 14.4 비관적 잠금(Pessimistic Locking)

### 간략 설명
데이터를 읽을 때 잠금을 걸어 다른 트랜잭션이 수정하지 못하게 합니다.

### SQL Server UPDLOCK

```csharp
// Raw SQL로 잠금 획득
using var transaction = await context.Database.BeginTransactionAsync();

try
{
    var product = await context.Products
        .FromSqlRaw("SELECT * FROM Products WITH (UPDLOCK) WHERE Id = {0}", productId)
        .FirstOrDefaultAsync();

    if (product != null)
    {
        product.Price = newPrice;
        await context.SaveChangesAsync();
    }

    await transaction.CommitAsync();
}
catch
{
    await transaction.RollbackAsync();
    throw;
}
```

### PostgreSQL FOR UPDATE

```csharp
var product = await context.Products
    .FromSqlRaw("SELECT * FROM \"Products\" WHERE \"Id\" = {0} FOR UPDATE", productId)
    .FirstOrDefaultAsync();
```

### 잠금 타임아웃

```csharp
// SQL Server
await context.Database.ExecuteSqlRawAsync("SET LOCK_TIMEOUT 5000");  // 5초

try
{
    var product = await context.Products
        .FromSqlRaw("SELECT * FROM Products WITH (UPDLOCK, ROWLOCK) WHERE Id = {0}", productId)
        .FirstOrDefaultAsync();
}
catch (SqlException ex) when (ex.Number == 1222)
{
    // 잠금 타임아웃
    throw new Exception("다른 사용자가 데이터를 수정 중입니다. 잠시 후 다시 시도해주세요.");
}
```

### 분산 잠금

```csharp
// Redis를 사용한 분산 잠금 예제
public class DistributedLockService
{
    private readonly IDatabase _redis;

    public async Task<bool> TryAcquireLockAsync(string resource, TimeSpan expiry)
    {
        var lockKey = $"lock:{resource}";
        return await _redis.StringSetAsync(lockKey, "1", expiry, When.NotExists);
    }

    public async Task ReleaseLockAsync(string resource)
    {
        var lockKey = $"lock:{resource}";
        await _redis.KeyDeleteAsync(lockKey);
    }
}

// 사용
public async Task UpdateProductAsync(int productId, decimal newPrice)
{
    var lockResource = $"product:{productId}";

    if (!await _lockService.TryAcquireLockAsync(lockResource, TimeSpan.FromSeconds(30)))
    {
        throw new Exception("다른 프로세스가 수정 중입니다.");
    }

    try
    {
        var product = await context.Products.FindAsync(productId);
        product.Price = newPrice;
        await context.SaveChangesAsync();
    }
    finally
    {
        await _lockService.ReleaseLockAsync(lockResource);
    }
}
```

---

## 실전 패턴

### 동시성 처리 서비스

```csharp
public interface IConcurrencyHandler<T> where T : class
{
    Task<T?> HandleConcurrencyAsync(DbUpdateConcurrencyException ex);
}

public class ProductConcurrencyHandler : IConcurrencyHandler<Product>
{
    private readonly IUserNotificationService _notificationService;

    public async Task<Product?> HandleConcurrencyAsync(DbUpdateConcurrencyException ex)
    {
        var entry = ex.Entries.OfType<EntityEntry<Product>>().FirstOrDefault();
        if (entry == null) return null;

        var databaseValues = await entry.GetDatabaseValuesAsync();
        if (databaseValues == null)
        {
            throw new InvalidOperationException("제품이 삭제되었습니다.");
        }

        var databaseProduct = (Product)databaseValues.ToObject();
        var proposedProduct = entry.Entity;

        // 가격 변경 충돌 시 사용자에게 알림
        if (databaseProduct.Price != proposedProduct.Price)
        {
            await _notificationService.NotifyAsync(
                $"가격이 {databaseProduct.Price}로 변경되었습니다. 다시 확인해주세요.");
        }

        return databaseProduct;
    }
}
```

### DTO에 버전 포함

```csharp
public class ProductUpdateDto
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public decimal Price { get; set; }
    public byte[] RowVersion { get; set; } = null!;  // 버전 포함
}

public async Task<bool> UpdateProductAsync(ProductUpdateDto dto)
{
    var product = new Product
    {
        Id = dto.Id,
        Name = dto.Name,
        Price = dto.Price,
        RowVersion = dto.RowVersion
    };

    context.Products.Update(product);

    try
    {
        await context.SaveChangesAsync();
        return true;
    }
    catch (DbUpdateConcurrencyException)
    {
        return false;
    }
}
```

---

## 요약

| 방식 | 사용 시나리오 | 장점 | 단점 |
|------|-------------|------|------|
| 낙관적 (RowVersion) | 충돌 드묾 | 확장성 | 재시도 필요 |
| 낙관적 (ConcurrencyCheck) | 특정 필드 | 세밀한 제어 | 설정 복잡 |
| 비관적 (UPDLOCK) | 충돌 빈번 | 충돌 방지 | 성능 저하 |
| 분산 잠금 | 마이크로서비스 | 서비스 간 조율 | 인프라 필요 |

## 다음 장 예고

다음 Part에서는 마이그레이션을 사용하여 데이터베이스 스키마를 관리하는 방법을 알아봅니다.
