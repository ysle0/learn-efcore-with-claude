# Chapter 36-40: 흔한 실수와 해결법

## 개요

EF Core 개발에서 자주 발생하는 실수들과 그 해결 방법을 알아봅니다. 각 실수의 원인을 이해하고 올바른 패턴을 학습합니다.

---

## 36. N+1 쿼리 문제

### 36.1 문제 상황

```csharp
// ❌ N+1 문제 발생
var orders = await context.Orders.ToListAsync();  // 1번 쿼리

foreach (var order in orders)
{
    // 각 주문마다 추가 쿼리 발생!
    Console.WriteLine($"Order {order.Id}: {order.Customer.Name}");  // N번 쿼리
}

// 결과: 100개 주문이면 101개 쿼리 실행!
```

### 36.2 원인

```
SELECT * FROM Orders                    -- 1번
SELECT * FROM Customers WHERE Id = 1    -- 2번
SELECT * FROM Customers WHERE Id = 2    -- 3번
...
SELECT * FROM Customers WHERE Id = 100  -- 101번
```

### 36.3 해결 방법

```csharp
// ✅ 해결책 1: Eager Loading (Include)
var orders = await context.Orders
    .Include(o => o.Customer)
    .ToListAsync();
// 결과: 1개 또는 2개 쿼리

// ✅ 해결책 2: Projection
var orderInfos = await context.Orders
    .Select(o => new
    {
        o.Id,
        o.OrderDate,
        CustomerName = o.Customer.Name
    })
    .ToListAsync();
// 결과: 1개 쿼리, JOIN 사용

// ✅ 해결책 3: Split Query (대량 데이터)
var orders = await context.Orders
    .Include(o => o.Customer)
    .Include(o => o.Items)
    .AsSplitQuery()
    .ToListAsync();
// 결과: 여러 개의 단순 쿼리 (Cartesian explosion 방지)
```

### 36.4 감지 방법

```csharp
// 로깅으로 N+1 감지
services.AddDbContext<ApplicationDbContext>(options =>
    options.UseSqlServer(connectionString)
           .LogTo(Console.WriteLine, LogLevel.Information)
           .EnableDetailedErrors());

// 또는 SQL Profiler, Application Insights 사용
```

---

## 37. 추적 관련 문제

### 37.1 동일 엔티티 중복 추적

```csharp
// ❌ 문제: 같은 ID의 엔티티를 다시 추가
var product1 = await context.Products.FindAsync(1);  // 추적됨

var product2 = new Product { Id = 1, Name = "Updated" };
context.Products.Update(product2);  // 오류! 이미 Id=1이 추적 중

// 오류 메시지:
// The instance of entity type 'Product' cannot be tracked because
// another instance with the key value 'Id:1' is already being tracked.
```

### 37.2 해결 방법

```csharp
// ✅ 해결책 1: 기존 엔티티 수정
var product = await context.Products.FindAsync(1);
product!.Name = "Updated";
await context.SaveChangesAsync();

// ✅ 해결책 2: Detach 후 Attach
var product1 = await context.Products.FindAsync(1);
context.Entry(product1!).State = EntityState.Detached;

var product2 = new Product { Id = 1, Name = "Updated" };
context.Products.Update(product2);
await context.SaveChangesAsync();

// ✅ 해결책 3: AsNoTracking으로 조회
var product = await context.Products
    .AsNoTracking()
    .FirstOrDefaultAsync(p => p.Id == 1);

// 이제 새 인스턴스로 업데이트 가능
var updateProduct = new Product { Id = 1, Name = "Updated" };
context.Products.Update(updateProduct);
await context.SaveChangesAsync();

// ✅ 해결책 4: ChangeTracker 초기화
context.ChangeTracker.Clear();
```

### 37.3 Detached 엔티티 수정

```csharp
// ❌ 문제: Detached 엔티티 직접 수정
var product = await context.Products.AsNoTracking().FirstAsync(p => p.Id == 1);
product.Price = 200;
await context.SaveChangesAsync();  // 변경 사항 저장 안 됨!

// ✅ 해결: Attach 또는 Update 사용
var product = await context.Products.AsNoTracking().FirstAsync(p => p.Id == 1);
product.Price = 200;
context.Products.Update(product);  // Modified 상태로 설정
await context.SaveChangesAsync();

// ✅ 또는 Entry 사용
context.Entry(product).State = EntityState.Modified;
await context.SaveChangesAsync();
```

---

## 38. Lazy Loading 함정

### 38.1 직렬화 시 무한 루프

```csharp
// ❌ 문제: 순환 참조로 인한 직렬화 오류
public class Order
{
    public int Id { get; set; }
    public virtual Customer Customer { get; set; } = null!;  // Lazy Load
}

public class Customer
{
    public int Id { get; set; }
    public virtual ICollection<Order> Orders { get; set; } = new List<Order>();  // Lazy Load
}

// 직렬화 시 Order → Customer → Orders → Order → ... 무한 루프!
var order = await context.Orders.FirstAsync();
return Json(order);  // 오류 또는 무한 루프!
```

### 38.2 해결 방법

```csharp
// ✅ 해결책 1: DTO 사용
var orderDto = await context.Orders
    .Select(o => new OrderDto
    {
        Id = o.Id,
        CustomerName = o.Customer.Name
        // Customer 객체 전체를 포함하지 않음
    })
    .FirstAsync();

// ✅ 해결책 2: JsonIgnore
public class Order
{
    public int Id { get; set; }

    [JsonIgnore]
    public virtual Customer Customer { get; set; } = null!;

    public int CustomerId { get; set; }  // FK만 노출
}

// ✅ 해결책 3: ReferenceHandler 설정
builder.Services.AddControllers()
    .AddJsonOptions(options =>
    {
        options.JsonSerializerOptions.ReferenceHandler = ReferenceHandler.IgnoreCycles;
    });
```

### 38.3 DbContext 해제 후 Lazy Loading

```csharp
// ❌ 문제: DbContext 해제 후 Navigation Property 접근
Order order;

using (var context = new ApplicationDbContext())
{
    order = await context.Orders.FirstAsync();
}

// DbContext가 이미 해제됨!
Console.WriteLine(order.Customer.Name);
// ObjectDisposedException 또는 null!
```

### 38.4 해결 방법

```csharp
// ✅ 해결책 1: 필요한 데이터 미리 로드
Order order;

using (var context = new ApplicationDbContext())
{
    order = await context.Orders
        .Include(o => o.Customer)
        .FirstAsync();
}

// 이제 안전하게 접근 가능
Console.WriteLine(order.Customer.Name);

// ✅ 해결책 2: Projection
var orderInfo = await context.Orders
    .Select(o => new
    {
        o.Id,
        CustomerName = o.Customer.Name
    })
    .FirstAsync();

// ✅ 해결책 3: Lazy Loading 비활성화
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
{
    optionsBuilder.UseLazyLoadingProxies(false);
}
```

---

## 39. 마이그레이션 실수

### 39.1 프로덕션에서 데이터 손실

```csharp
// ❌ 위험: 컬럼 타입 변경
public class Product
{
    public int Id { get; set; }
    // public string Description { get; set; }  // 이전
    public int Description { get; set; }  // 변경 - 데이터 손실!
}

// ❌ 위험: 컬럼 삭제
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    // public string Description { get; set; }  // 삭제됨 - 데이터 손실!
}
```

### 39.2 해결 방법

```csharp
// ✅ 안전한 마이그레이션: 단계적 접근

// Step 1: 새 컬럼 추가
public class Product
{
    public int Id { get; set; }
    public string? OldDescription { get; set; }  // 기존 (nullable로 변경)
    public string? NewDescription { get; set; }  // 신규
}

// Step 2: 데이터 마이그레이션 (별도 스크립트)
await context.Database.ExecuteSqlRawAsync(@"
    UPDATE Products
    SET NewDescription = OldDescription
    WHERE NewDescription IS NULL
");

// Step 3: 새 컬럼 NOT NULL로 변경 (다음 마이그레이션)

// Step 4: 이전 컬럼 삭제 (충분한 검증 후)
```

### 39.3 마이그레이션 충돌

```bash
# ❌ 문제: 여러 개발자가 동시에 마이그레이션 생성
Developer A: 20240101_AddProductColumn
Developer B: 20240102_AddCategoryColumn

# 병합 시 충돌 발생
```

### 39.4 해결 방법

```bash
# ✅ 해결: 충돌 마이그레이션 다시 생성

# 1. 충돌 마이그레이션 삭제
rm -rf Migrations/20240101_AddProductColumn.cs
rm -rf Migrations/20240102_AddCategoryColumn.cs

# 2. 스냅샷 업데이트 후 새 마이그레이션 생성
dotnet ef migrations add CombinedChanges

# 3. 마이그레이션 검토 후 적용
dotnet ef database update
```

### 39.5 안전한 마이그레이션 워크플로우

```csharp
// 마이그레이션 전 체크리스트

// 1. 백업 확인
// 2. 마이그레이션 스크립트 검토
var script = context.Database.GenerateCreateScript();

// 3. 스테이징 환경에서 먼저 테스트

// 4. 프로덕션 마이그레이션 시 트랜잭션 스크립트 사용
// dotnet ef migrations script --idempotent -o migrate.sql
```

---

## 40. 기타 흔한 실수

### 40.1 SaveChanges 호출 누락

```csharp
// ❌ 문제: SaveChanges 호출 누락
var product = await context.Products.FindAsync(1);
product!.Price = 200;
// await context.SaveChangesAsync();  // 누락!

// 변경 사항이 저장되지 않음!
```

### 40.2 비동기 메서드 오용

```csharp
// ❌ 문제: async void
public async void SaveProductAsync(Product product)
{
    context.Products.Add(product);
    await context.SaveChangesAsync();
    // 예외가 발생해도 호출자에게 전파되지 않음!
}

// ✅ 올바른 방법
public async Task SaveProductAsync(Product product)
{
    context.Products.Add(product);
    await context.SaveChangesAsync();
}

// ❌ 문제: 비동기 메서드를 동기로 호출
var products = context.Products.ToListAsync().Result;  // 데드락 위험!
var products = context.Products.ToListAsync().GetAwaiter().GetResult();  // 위험!

// ✅ 올바른 방법
var products = await context.Products.ToListAsync();
```

### 40.3 DbContext 스레드 안전성

```csharp
// ❌ 문제: 여러 스레드에서 동일 DbContext 사용
var tasks = productIds.Select(async id =>
{
    var product = await context.Products.FindAsync(id);  // 스레드 안전하지 않음!
    product!.Price *= 1.1m;
});
await Task.WhenAll(tasks);
```

### 40.4 해결 방법

```csharp
// ✅ 해결: 각 작업에 별도 DbContext
var tasks = productIds.Select(async id =>
{
    await using var context = await contextFactory.CreateDbContextAsync();
    var product = await context.Products.FindAsync(id);
    product!.Price *= 1.1m;
    await context.SaveChangesAsync();
});
await Task.WhenAll(tasks);

// ✅ 또는 순차 처리
foreach (var id in productIds)
{
    var product = await context.Products.FindAsync(id);
    product!.Price *= 1.1m;
}
await context.SaveChangesAsync();

// ✅ 또는 ExecuteUpdate 사용
await context.Products
    .Where(p => productIds.Contains(p.Id))
    .ExecuteUpdateAsync(s => s.SetProperty(p => p.Price, p => p.Price * 1.1m));
```

### 40.5 Include 과다 사용

```csharp
// ❌ 문제: 불필요한 데이터 로딩
var order = await context.Orders
    .Include(o => o.Customer)
        .ThenInclude(c => c.Address)
        .ThenInclude(a => a.Country)
    .Include(o => o.Items)
        .ThenInclude(i => i.Product)
            .ThenInclude(p => p.Category)
    .Include(o => o.Payments)
    .Include(o => o.ShippingInfo)
    .FirstOrDefaultAsync(o => o.Id == orderId);

// 필요한 건 주문 번호와 고객 이름뿐인데...

// ✅ 해결: Projection 사용
var orderInfo = await context.Orders
    .Where(o => o.Id == orderId)
    .Select(o => new
    {
        o.OrderNumber,
        CustomerName = o.Customer.Name
    })
    .FirstOrDefaultAsync();
```

### 40.6 글로벌 필터 무시

```csharp
// 글로벌 필터 설정
modelBuilder.Entity<Product>()
    .HasQueryFilter(p => !p.IsDeleted);

// ❌ 문제: 필터가 적용되어 예상과 다른 결과
var product = await context.Products.FindAsync(deletedProductId);
// null 반환! (삭제된 상품)

// ✅ 해결: 필터 무시가 필요한 경우
var product = await context.Products
    .IgnoreQueryFilters()
    .FirstOrDefaultAsync(p => p.Id == deletedProductId);
```

---

## 실수 방지 체크리스트

### 쿼리 관련
- [ ] N+1 문제 확인 (Include 또는 Projection 사용)
- [ ] 필요한 데이터만 조회 (Select 사용)
- [ ] 읽기 전용에 AsNoTracking 사용
- [ ] 페이지네이션에 OrderBy 포함

### 변경 추적 관련
- [ ] SaveChanges 호출 확인
- [ ] 동일 엔티티 중복 추적 방지
- [ ] Detached 엔티티 처리 확인

### Lazy Loading 관련
- [ ] DbContext 수명 내 접근 확인
- [ ] 직렬화 시 순환 참조 방지
- [ ] 필요한 데이터 미리 로드

### 마이그레이션 관련
- [ ] 데이터 손실 가능성 검토
- [ ] 프로덕션 적용 전 테스트
- [ ] 롤백 계획 수립

### 기타
- [ ] 비동기 메서드 올바르게 사용
- [ ] DbContext 스레드 안전성 확인
- [ ] 글로벌 필터 영향 확인

---

## 요약

| 실수 | 원인 | 해결책 |
|------|------|--------|
| N+1 쿼리 | Lazy Loading | Include, Projection |
| 중복 추적 | 같은 ID 엔티티 | Detach, AsNoTracking |
| 직렬화 루프 | 순환 참조 | DTO, JsonIgnore |
| 데이터 손실 | 마이그레이션 | 단계적 마이그레이션 |
| 데드락 | 동기 호출 | async/await 사용 |

## 다음 Part 예고

다음 Part에서는 EF Core의 심화 고급 주제들을 알아봅니다.
