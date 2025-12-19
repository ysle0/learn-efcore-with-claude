# Chapter 12: 변경 추적(Change Tracking)

## 개요

EF Core의 변경 추적(Change Tracking) 시스템이 어떻게 동작하는지 알아봅니다. 엔티티 상태, 변경 감지, ChangeTracker API를 상세히 다룹니다.

---

## 12.1 엔티티 상태(Entity States)

### 간략 설명
EF Core는 각 엔티티의 상태를 추적하여 SaveChanges 호출 시 적절한 SQL을 생성합니다.

### 5가지 엔티티 상태

```csharp
public enum EntityState
{
    Detached,    // 추적되지 않음
    Unchanged,   // 추적 중, 변경 없음
    Added,       // 새로 추가됨 (INSERT 예정)
    Modified,    // 수정됨 (UPDATE 예정)
    Deleted      // 삭제됨 (DELETE 예정)
}
```

### 상태 전이 다이어그램

```
                      ┌──────────────────┐
                      │     Detached     │
                      │  (추적 안 됨)     │
                      └────────┬─────────┘
                               │
          Attach()/Update()    │    Add()
          ┌────────────────────┼────────────────────┐
          │                    │                    │
          ▼                    ▼                    ▼
    ┌───────────┐        ┌───────────┐        ┌───────────┐
    │ Unchanged │◄──────►│  Modified │        │   Added   │
    │ (변경없음) │ 속성수정 │  (수정됨)  │        │ (추가됨)  │
    └─────┬─────┘        └─────┬─────┘        └─────┬─────┘
          │                    │                    │
          │     Remove()       │                    │
          ▼                    ▼                    │
    ┌───────────┐              │                    │
    │  Deleted  │◄─────────────┘                    │
    │  (삭제됨)  │                                   │
    └─────┬─────┘                                   │
          │                                         │
          └─────────────SaveChanges()───────────────┘
                               │
                               ▼
                        ┌───────────┐
                        │ Unchanged │ (또는 Detached)
                        └───────────┘
```

### 상태 확인 및 변경

```csharp
// 상태 확인
var entry = context.Entry(product);
Console.WriteLine($"상태: {entry.State}");

// 상태 직접 변경
context.Entry(product).State = EntityState.Modified;

// 상태별 동작
var product = new Product { Id = 1, Name = "Test" };

// Detached → Added
context.Products.Add(product);
Console.WriteLine(context.Entry(product).State);  // Added

// Detached → Unchanged (기존 데이터 가정)
context.Products.Attach(product);
Console.WriteLine(context.Entry(product).State);  // Unchanged

// Detached → Modified
context.Products.Update(product);
Console.WriteLine(context.Entry(product).State);  // Modified

// Unchanged/Modified → Deleted
context.Products.Remove(product);
Console.WriteLine(context.Entry(product).State);  // Deleted
```

### 각 상태별 SaveChanges 동작

```csharp
// Added → INSERT
var newProduct = new Product { Name = "New Product", Price = 100 };
context.Products.Add(newProduct);
await context.SaveChangesAsync();
// SQL: INSERT INTO Products (Name, Price) VALUES ('New Product', 100)

// Modified → UPDATE
var product = await context.Products.FindAsync(1);
product.Price = 150;
await context.SaveChangesAsync();
// SQL: UPDATE Products SET Price = 150 WHERE Id = 1

// Deleted → DELETE
var productToDelete = await context.Products.FindAsync(2);
context.Products.Remove(productToDelete);
await context.SaveChangesAsync();
// SQL: DELETE FROM Products WHERE Id = 2

// Unchanged → 아무 것도 안 함
var unchangedProduct = await context.Products.FindAsync(3);
await context.SaveChangesAsync();
// SQL: (없음)
```

---

## 12.2 AsNoTracking 사용

### 간략 설명
읽기 전용 쿼리에서 변경 추적을 비활성화하여 성능을 향상시킵니다.

### 기본 사용법

```csharp
// 변경 추적 비활성화
var products = await context.Products
    .AsNoTracking()
    .ToListAsync();

// 상태 확인
foreach (var product in products)
{
    Console.WriteLine(context.Entry(product).State);  // Detached
}

// 수정해도 추적되지 않음
products[0].Price = 999;
await context.SaveChangesAsync();  // 아무 일도 안 일어남
```

### AsNoTracking의 장점

```csharp
// 1. 메모리 절약
// 추적 정보를 저장하지 않음

// 2. 성능 향상
// 스냅샷 생성, 변경 감지 오버헤드 없음

// 3. 대량 조회에 적합
var allProducts = await context.Products
    .AsNoTracking()
    .ToListAsync();  // 수천 개의 제품 조회 시 유리
```

### AsNoTrackingWithIdentityResolution

```csharp
// 문제: AsNoTracking은 같은 엔티티를 여러 인스턴스로 반환
var posts = await context.Posts
    .AsNoTracking()
    .Include(p => p.Author)
    .ToListAsync();

// 같은 Author라도 다른 인스턴스
var author1 = posts[0].Author;
var author2 = posts[1].Author;
Console.WriteLine(ReferenceEquals(author1, author2));  // false (같은 ID라도)

// 해결책: AsNoTrackingWithIdentityResolution (EF Core 5+)
var posts = await context.Posts
    .AsNoTrackingWithIdentityResolution()
    .Include(p => p.Author)
    .ToListAsync();

Console.WriteLine(ReferenceEquals(author1, author2));  // true (같은 ID면)
```

### 전역 AsNoTracking 설정

```csharp
// DbContext 수준에서 기본 동작 변경
public class ReadOnlyDbContext : DbContext
{
    public ReadOnlyDbContext(DbContextOptions<ReadOnlyDbContext> options)
        : base(options)
    {
        ChangeTracker.QueryTrackingBehavior = QueryTrackingBehavior.NoTracking;
    }
}

// 또는 생성자에서
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
{
    optionsBuilder.UseQueryTrackingBehavior(QueryTrackingBehavior.NoTracking);
}

// 특정 쿼리에서 추적 활성화
var product = await context.Products
    .AsTracking()
    .FirstOrDefaultAsync(p => p.Id == 1);
```

---

## 12.3 변경 감지 전략

### 간략 설명
EF Core가 엔티티의 변경을 감지하는 방법과 이를 최적화하는 전략을 알아봅니다.

### 스냅샷 변경 감지 (기본)

```csharp
// 기본 동작: 스냅샷 비교
var product = await context.Products.FindAsync(1);
// EF Core가 원본 값의 스냅샷 저장

product.Price = 150;  // 값 변경

// DetectChanges 호출 시 스냅샷과 현재 값 비교
context.ChangeTracker.DetectChanges();

// SaveChanges 전에 자동으로 DetectChanges 호출됨
await context.SaveChangesAsync();
```

### DetectChanges 호출 시점

```csharp
// 자동 호출되는 메서드들
await context.SaveChangesAsync();     // DetectChanges 호출됨
context.Entry(entity);                // DetectChanges 호출됨
context.ChangeTracker.Entries();      // DetectChanges 호출됨

// 수동 호출
context.ChangeTracker.DetectChanges();
```

### 자동 감지 비활성화

```csharp
// 대량 작업 시 성능 최적화
context.ChangeTracker.AutoDetectChangesEnabled = false;

try
{
    foreach (var product in products)
    {
        product.Price *= 1.1m;
        // 상태를 명시적으로 설정
        context.Entry(product).State = EntityState.Modified;
    }

    await context.SaveChangesAsync();
}
finally
{
    context.ChangeTracker.AutoDetectChangesEnabled = true;
}
```

### 프록시 변경 추적

```csharp
// 프록시 기반 변경 추적 (알림 방식)
// INotifyPropertyChanged 구현 필요

public class Product : INotifyPropertyChanged
{
    private string _name = string.Empty;
    private decimal _price;

    public event PropertyChangedEventHandler? PropertyChanged;

    public int Id { get; set; }

    public string Name
    {
        get => _name;
        set
        {
            if (_name != value)
            {
                _name = value;
                OnPropertyChanged();
            }
        }
    }

    public decimal Price
    {
        get => _price;
        set
        {
            if (_price != value)
            {
                _price = value;
                OnPropertyChanged();
            }
        }
    }

    protected void OnPropertyChanged([CallerMemberName] string? propertyName = null)
    {
        PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(propertyName));
    }
}

// 설정
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
{
    optionsBuilder.UseChangeTrackingProxies();  // 패키지 필요
}
```

---

## 12.4 ChangeTracker API

### 간략 설명
ChangeTracker API를 사용하여 추적 중인 엔티티를 검사하고 조작하는 방법을 알아봅니다.

### Entries 조회

```csharp
// 모든 추적 중인 엔티티
var allEntries = context.ChangeTracker.Entries();

foreach (var entry in allEntries)
{
    Console.WriteLine($"Type: {entry.Entity.GetType().Name}");
    Console.WriteLine($"State: {entry.State}");
}

// 특정 타입만
var productEntries = context.ChangeTracker.Entries<Product>();

// 특정 상태만
var modifiedEntries = context.ChangeTracker.Entries()
    .Where(e => e.State == EntityState.Modified);

var addedAndModified = context.ChangeTracker.Entries()
    .Where(e => e.State == EntityState.Added || e.State == EntityState.Modified);
```

### 속성 수준 접근

```csharp
var product = await context.Products.FindAsync(1);
product.Price = 150;
product.Name = "Updated Name";

var entry = context.Entry(product);

// 모든 속성 반복
foreach (var property in entry.Properties)
{
    Console.WriteLine($"Property: {property.Metadata.Name}");
    Console.WriteLine($"  Original: {property.OriginalValue}");
    Console.WriteLine($"  Current: {property.CurrentValue}");
    Console.WriteLine($"  Modified: {property.IsModified}");
}

// 특정 속성 접근
var priceProperty = entry.Property(p => p.Price);
Console.WriteLine($"Original Price: {priceProperty.OriginalValue}");
Console.WriteLine($"Current Price: {priceProperty.CurrentValue}");

// 원본 값으로 복원
priceProperty.CurrentValue = priceProperty.OriginalValue;
priceProperty.IsModified = false;
```

### 변경 취소

```csharp
// 단일 엔티티 변경 취소
var entry = context.Entry(product);
entry.CurrentValues.SetValues(entry.OriginalValues);
entry.State = EntityState.Unchanged;

// 모든 변경 취소
foreach (var entry in context.ChangeTracker.Entries())
{
    switch (entry.State)
    {
        case EntityState.Modified:
            entry.CurrentValues.SetValues(entry.OriginalValues);
            entry.State = EntityState.Unchanged;
            break;
        case EntityState.Added:
            entry.State = EntityState.Detached;
            break;
        case EntityState.Deleted:
            entry.State = EntityState.Unchanged;
            break;
    }
}

// 또는 간단히
context.ChangeTracker.Clear();  // 모든 추적 중지
```

### 감사 로그 구현

```csharp
public interface IAuditable
{
    DateTime CreatedAt { get; set; }
    DateTime? ModifiedAt { get; set; }
    string CreatedBy { get; set; }
    string? ModifiedBy { get; set; }
}

public class AuditableDbContext : DbContext
{
    private readonly ICurrentUserService _currentUser;

    public override async Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
    {
        var now = DateTime.UtcNow;
        var userId = _currentUser.UserId;

        foreach (var entry in ChangeTracker.Entries<IAuditable>())
        {
            switch (entry.State)
            {
                case EntityState.Added:
                    entry.Entity.CreatedAt = now;
                    entry.Entity.CreatedBy = userId;
                    break;

                case EntityState.Modified:
                    entry.Entity.ModifiedAt = now;
                    entry.Entity.ModifiedBy = userId;
                    // CreatedAt, CreatedBy는 수정 안 함
                    entry.Property(x => x.CreatedAt).IsModified = false;
                    entry.Property(x => x.CreatedBy).IsModified = false;
                    break;
            }
        }

        return await base.SaveChangesAsync(cancellationToken);
    }
}
```

### 변경 히스토리 기록

```csharp
public class AuditLog
{
    public int Id { get; set; }
    public string EntityName { get; set; } = string.Empty;
    public string EntityId { get; set; } = string.Empty;
    public string Action { get; set; } = string.Empty;
    public string Changes { get; set; } = string.Empty;
    public DateTime Timestamp { get; set; }
    public string UserId { get; set; } = string.Empty;
}

public override async Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
{
    var auditLogs = new List<AuditLog>();

    foreach (var entry in ChangeTracker.Entries())
    {
        if (entry.State == EntityState.Detached || entry.State == EntityState.Unchanged)
            continue;

        var auditLog = new AuditLog
        {
            EntityName = entry.Entity.GetType().Name,
            EntityId = GetPrimaryKeyValue(entry),
            Action = entry.State.ToString(),
            Timestamp = DateTime.UtcNow,
            UserId = _currentUser.UserId
        };

        if (entry.State == EntityState.Modified)
        {
            var changes = new Dictionary<string, object?>();
            foreach (var property in entry.Properties.Where(p => p.IsModified))
            {
                changes[property.Metadata.Name] = new
                {
                    Old = property.OriginalValue,
                    New = property.CurrentValue
                };
            }
            auditLog.Changes = JsonSerializer.Serialize(changes);
        }

        auditLogs.Add(auditLog);
    }

    // 감사 로그 저장
    AuditLogs.AddRange(auditLogs);

    return await base.SaveChangesAsync(cancellationToken);
}

private string GetPrimaryKeyValue(EntityEntry entry)
{
    var keyProperties = entry.Metadata.FindPrimaryKey()?.Properties;
    if (keyProperties == null) return string.Empty;

    var values = keyProperties.Select(p => entry.Property(p.Name).CurrentValue);
    return string.Join(",", values);
}
```

---

## 요약

| 개념 | 설명 | 사용 시나리오 |
|------|------|-------------|
| EntityState | 엔티티의 현재 상태 | SaveChanges 시 SQL 결정 |
| AsNoTracking | 추적 비활성화 | 읽기 전용 쿼리 |
| DetectChanges | 변경 감지 | 자동/수동 호출 |
| ChangeTracker | 추적 API | 감사, 변경 취소 |

## 다음 장 예고

다음 장에서는 Add, Update, Remove와 SaveChanges를 사용한 데이터 저장 방법을 알아봅니다.
