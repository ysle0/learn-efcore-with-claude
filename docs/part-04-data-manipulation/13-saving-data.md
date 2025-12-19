# Chapter 13: 데이터 저장

## 개요

EF Core에서 데이터를 추가, 수정, 삭제하고 저장하는 방법을 알아봅니다. 개별 작업부터 배치 작업, 트랜잭션까지 다룹니다.

---

## 13.1 Add, Update, Remove

### Add - 데이터 추가

```csharp
// 단일 엔티티 추가
var product = new Product
{
    Name = "New Product",
    Price = 100,
    CategoryId = 1
};

context.Products.Add(product);
await context.SaveChangesAsync();

// ID가 자동으로 설정됨
Console.WriteLine($"생성된 ID: {product.Id}");

// DbContext.Add 사용 (동일)
context.Add(product);

// 비동기 버전
await context.Products.AddAsync(product);
// 참고: AddAsync는 특수한 값 생성기가 있을 때만 유용
// 일반적으로 Add 사용 권장
```

### 관련 데이터와 함께 추가

```csharp
// 새 블로그와 포스트 함께 추가
var blog = new Blog
{
    Name = "Tech Blog",
    Url = "https://tech.example.com",
    Posts = new List<Post>
    {
        new Post { Title = "First Post", Content = "Hello World" },
        new Post { Title = "Second Post", Content = "More content" }
    }
};

context.Blogs.Add(blog);
await context.SaveChangesAsync();
// Blog와 모든 Posts가 INSERT됨

// 기존 엔티티에 관련 데이터 추가
var existingBlog = await context.Blogs.FindAsync(1);
existingBlog.Posts.Add(new Post { Title = "New Post", Content = "Content" });
await context.SaveChangesAsync();
// 새 Post만 INSERT됨
```

### Update - 데이터 수정

```csharp
// 방법 1: 추적 중인 엔티티 수정 (권장)
var product = await context.Products.FindAsync(1);
product.Price = 150;
product.Name = "Updated Name";
await context.SaveChangesAsync();
// 변경된 속성만 UPDATE

// 생성되는 SQL
// UPDATE Products SET Price = 150, Name = 'Updated Name' WHERE Id = 1

// 방법 2: Attach + 수정
var detachedProduct = new Product { Id = 1 };
context.Products.Attach(detachedProduct);
detachedProduct.Price = 200;
await context.SaveChangesAsync();
// Price만 UPDATE

// 방법 3: Update (모든 속성 UPDATE)
var productToUpdate = new Product
{
    Id = 1,
    Name = "Full Update",
    Price = 300,
    Description = "Updated description"
};
context.Products.Update(productToUpdate);
await context.SaveChangesAsync();
// 모든 컬럼이 UPDATE됨 (비효율적일 수 있음)
```

### Remove - 데이터 삭제

```csharp
// 방법 1: 추적 중인 엔티티 삭제
var product = await context.Products.FindAsync(1);
context.Products.Remove(product);
await context.SaveChangesAsync();

// 방법 2: ID만으로 삭제 (조회 없이)
var productToDelete = new Product { Id = 1 };
context.Products.Attach(productToDelete);
context.Products.Remove(productToDelete);
await context.SaveChangesAsync();

// 또는
context.Entry(new Product { Id = 1 }).State = EntityState.Deleted;
await context.SaveChangesAsync();

// 관련 데이터 삭제 (Cascade)
var blog = await context.Blogs
    .Include(b => b.Posts)
    .FirstAsync(b => b.Id == 1);

context.Blogs.Remove(blog);
await context.SaveChangesAsync();
// Blog와 모든 Posts가 DELETE됨 (Cascade 설정에 따라)
```

---

## 13.2 SaveChanges와 SaveChangesAsync

### 기본 동작

```csharp
// 동기 버전 (비권장)
int affectedRows = context.SaveChanges();

// 비동기 버전 (권장)
int affectedRows = await context.SaveChangesAsync();

// 취소 토큰 사용
var cts = new CancellationTokenSource();
int affectedRows = await context.SaveChangesAsync(cts.Token);
```

### SaveChanges 동작 원리

```csharp
// SaveChanges는 다음을 수행:
// 1. DetectChanges 호출
// 2. 유효성 검사 (선택적)
// 3. 인터셉터 실행
// 4. SQL 생성 및 실행
// 5. 생성된 값 다시 읽기 (IDENTITY 등)
// 6. 상태 변경 (Modified → Unchanged 등)

var product = new Product { Name = "Test", Price = 100 };
context.Products.Add(product);
Console.WriteLine($"Before: {product.Id}");  // 0

await context.SaveChangesAsync();
Console.WriteLine($"After: {product.Id}");   // 자동 생성된 ID
```

### SaveChanges 반환 값

```csharp
// 영향받은 행 수 반환
var product1 = new Product { Name = "P1", Price = 100 };
var product2 = new Product { Name = "P2", Price = 200 };

context.Products.AddRange(product1, product2);
int count = await context.SaveChangesAsync();
Console.WriteLine($"Affected rows: {count}");  // 2
```

### 자동 생성 값 처리

```csharp
public class Order
{
    public int Id { get; set; }  // DB에서 자동 생성
    public string OrderNumber { get; set; } = string.Empty;
    public DateTime CreatedAt { get; set; }  // DB 기본값

    [DatabaseGenerated(DatabaseGeneratedOption.Computed)]
    public decimal TotalWithTax { get; set; }  // 계산 컬럼
}

// 설정
modelBuilder.Entity<Order>(entity =>
{
    entity.Property(o => o.CreatedAt)
        .HasDefaultValueSql("GETUTCDATE()");

    entity.Property(o => o.TotalWithTax)
        .HasComputedColumnSql("[Total] * 1.1");
});

// 사용
var order = new Order { OrderNumber = "ORD-001" };
context.Orders.Add(order);
await context.SaveChangesAsync();

// DB에서 생성된 값이 엔티티에 반영됨
Console.WriteLine($"Id: {order.Id}");
Console.WriteLine($"CreatedAt: {order.CreatedAt}");
Console.WriteLine($"TotalWithTax: {order.TotalWithTax}");
```

---

## 13.3 AddRange, UpdateRange, RemoveRange

### 대량 추가

```csharp
// AddRange
var products = new List<Product>
{
    new Product { Name = "Product 1", Price = 100 },
    new Product { Name = "Product 2", Price = 200 },
    new Product { Name = "Product 3", Price = 300 }
};

context.Products.AddRange(products);
await context.SaveChangesAsync();

// 또는 params 사용
context.Products.AddRange(
    new Product { Name = "P1" },
    new Product { Name = "P2" }
);
```

### 대량 수정

```csharp
// UpdateRange - 모든 속성 UPDATE
var productsToUpdate = new List<Product>
{
    new Product { Id = 1, Name = "Updated 1", Price = 110 },
    new Product { Id = 2, Name = "Updated 2", Price = 220 }
};

context.Products.UpdateRange(productsToUpdate);
await context.SaveChangesAsync();

// 주의: 모든 컬럼이 UPDATE됨
```

### 대량 삭제

```csharp
// RemoveRange
var productsToDelete = await context.Products
    .Where(p => p.IsDiscontinued)
    .ToListAsync();

context.Products.RemoveRange(productsToDelete);
await context.SaveChangesAsync();

// ID만으로 삭제
var idsToDelete = new[] { 1, 2, 3 };
var productsToDelete = idsToDelete
    .Select(id => new Product { Id = id });

context.Products.RemoveRange(productsToDelete);
await context.SaveChangesAsync();
```

### 배치 처리 최적화

```csharp
// 기본 SaveChanges는 개별 INSERT 실행
// SQL Server: INSERT ... VALUES (...); INSERT ... VALUES (...); ...

// 배치 크기 설정 (EF Core가 자동 배치)
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
{
    optionsBuilder.UseSqlServer(connectionString, options =>
    {
        options.MaxBatchSize(100);  // 기본값: 42
        options.MinBatchSize(2);    // 기본값: 4
    });
}

// 대량 삽입 시 성능 팁
context.ChangeTracker.AutoDetectChangesEnabled = false;

var products = GenerateManyProducts(10000);
context.Products.AddRange(products);
await context.SaveChangesAsync();

context.ChangeTracker.AutoDetectChangesEnabled = true;
```

---

## 13.4 트랜잭션 처리

### 기본 트랜잭션 (SaveChanges)

```csharp
// SaveChanges는 자동으로 트랜잭션 사용
try
{
    context.Products.Add(new Product { Name = "P1" });
    context.Products.Add(new Product { Name = "P2" });
    await context.SaveChangesAsync();  // 모두 성공하거나 모두 롤백
}
catch (Exception)
{
    // 모든 변경이 롤백됨
}
```

### 명시적 트랜잭션

```csharp
// 여러 SaveChanges를 하나의 트랜잭션으로
using var transaction = await context.Database.BeginTransactionAsync();

try
{
    var order = new Order { CustomerId = 1 };
    context.Orders.Add(order);
    await context.SaveChangesAsync();  // Order 저장

    foreach (var item in orderItems)
    {
        item.OrderId = order.Id;
        context.OrderItems.Add(item);
    }
    await context.SaveChangesAsync();  // OrderItems 저장

    await transaction.CommitAsync();
}
catch (Exception)
{
    await transaction.RollbackAsync();
    throw;
}
```

### 세이브포인트

```csharp
using var transaction = await context.Database.BeginTransactionAsync();

try
{
    // 첫 번째 작업
    context.Orders.Add(order1);
    await context.SaveChangesAsync();

    // 세이브포인트 생성
    await transaction.CreateSavepointAsync("AfterOrder1");

    try
    {
        // 두 번째 작업 (실패해도 첫 번째 유지)
        context.Orders.Add(order2);
        await context.SaveChangesAsync();
    }
    catch
    {
        // 세이브포인트로 롤백
        await transaction.RollbackToSavepointAsync("AfterOrder1");
        // order1은 유지됨
    }

    await transaction.CommitAsync();
}
catch
{
    await transaction.RollbackAsync();
}
```

### 여러 DbContext 트랜잭션

```csharp
// 같은 연결 공유
using var connection = new SqlConnection(connectionString);
await connection.OpenAsync();

using var transaction = await connection.BeginTransactionAsync();

try
{
    var options1 = new DbContextOptionsBuilder<OrderContext>()
        .UseSqlServer(connection)
        .Options;

    var options2 = new DbContextOptionsBuilder<InventoryContext>()
        .UseSqlServer(connection)
        .Options;

    using var orderContext = new OrderContext(options1);
    using var inventoryContext = new InventoryContext(options2);

    orderContext.Database.UseTransaction(transaction as DbTransaction);
    inventoryContext.Database.UseTransaction(transaction as DbTransaction);

    // 주문 생성
    orderContext.Orders.Add(new Order { ... });
    await orderContext.SaveChangesAsync();

    // 재고 감소
    var inventory = await inventoryContext.Inventories.FindAsync(productId);
    inventory.Quantity -= orderQuantity;
    await inventoryContext.SaveChangesAsync();

    await transaction.CommitAsync();
}
catch
{
    await transaction.RollbackAsync();
}
```

### TransactionScope (분산 트랜잭션)

```csharp
// 주의: 분산 트랜잭션은 DTC 필요
using var scope = new TransactionScope(
    TransactionScopeOption.Required,
    new TransactionOptions { IsolationLevel = IsolationLevel.ReadCommitted },
    TransactionScopeAsyncFlowOption.Enabled);

try
{
    using var context1 = new Database1Context();
    context1.Entities.Add(new Entity1());
    await context1.SaveChangesAsync();

    using var context2 = new Database2Context();
    context2.Entities.Add(new Entity2());
    await context2.SaveChangesAsync();

    scope.Complete();
}
catch
{
    // TransactionScope 자동 롤백
}
```

### 격리 수준

```csharp
// 격리 수준 설정
using var transaction = await context.Database
    .BeginTransactionAsync(IsolationLevel.Serializable);

// 격리 수준 옵션
IsolationLevel.ReadUncommitted  // Dirty Read 허용
IsolationLevel.ReadCommitted    // 기본값
IsolationLevel.RepeatableRead   // 반복 읽기 보장
IsolationLevel.Serializable     // 완전 격리 (가장 느림)
IsolationLevel.Snapshot         // 스냅샷 격리 (SQL Server)
```

---

## ExecuteUpdate/ExecuteDelete (EF Core 7+)

### 대량 업데이트

```csharp
// 조회 없이 직접 UPDATE
await context.Products
    .Where(p => p.CategoryId == 5)
    .ExecuteUpdateAsync(setters => setters
        .SetProperty(p => p.Price, p => p.Price * 1.1m)
        .SetProperty(p => p.LastModified, DateTime.UtcNow));

// SQL: UPDATE Products SET Price = Price * 1.1, LastModified = @now WHERE CategoryId = 5

// 조건부 업데이트
await context.Products
    .Where(p => p.Stock < 10)
    .ExecuteUpdateAsync(s => s
        .SetProperty(p => p.Status, ProductStatus.LowStock));
```

### 대량 삭제

```csharp
// 조회 없이 직접 DELETE
await context.Products
    .Where(p => p.IsDiscontinued)
    .ExecuteDeleteAsync();

// SQL: DELETE FROM Products WHERE IsDiscontinued = 1

// 오래된 로그 삭제
await context.Logs
    .Where(l => l.CreatedAt < DateTime.UtcNow.AddMonths(-6))
    .ExecuteDeleteAsync();
```

### 주의사항

```csharp
// ExecuteUpdate/ExecuteDelete는:
// - ChangeTracker를 거치지 않음
// - 인터셉터/이벤트 발생 안 함
// - Soft Delete 패턴 우회
// - 캐시된 엔티티와 동기화 안 됨

// 사용 후 캐시 초기화 필요할 수 있음
await context.Products
    .Where(p => p.CategoryId == 5)
    .ExecuteUpdateAsync(s => s.SetProperty(p => p.Price, 100));

context.ChangeTracker.Clear();  // 캐시 초기화
```

---

## 요약

| 메서드 | 용도 | 특징 |
|--------|------|------|
| Add | 삽입 | 상태 → Added |
| Update | 전체 수정 | 모든 컬럼 UPDATE |
| Remove | 삭제 | 상태 → Deleted |
| SaveChanges | 저장 | 트랜잭션 자동 |
| ExecuteUpdate | 대량 수정 | ChangeTracker 우회 |
| ExecuteDelete | 대량 삭제 | ChangeTracker 우회 |

## 다음 장 예고

다음 장에서는 동시성 제어를 통해 여러 사용자가 동시에 데이터를 수정할 때 발생하는 충돌을 처리하는 방법을 알아봅니다.
