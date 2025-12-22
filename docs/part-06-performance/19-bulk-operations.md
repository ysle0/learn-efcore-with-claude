# Chapter 19: 벌크 작업

## 개요

대량의 데이터를 효율적으로 처리하는 방법을 알아봅니다. EF Core 7+의 ExecuteUpdate/Delete와 서드파티 라이브러리를 다룹니다.

---

## 18.1 ExecuteUpdate (EF Core 7+)

### 간략 설명
ChangeTracker를 거치지 않고 직접 UPDATE SQL을 실행합니다.

### 기본 사용법

```csharp
// 가격 10% 인상
await context.Products
    .Where(p => p.CategoryId == 5)
    .ExecuteUpdateAsync(setters => setters
        .SetProperty(p => p.Price, p => p.Price * 1.1m));

// 생성되는 SQL:
// UPDATE [Products] SET [Price] = [Price] * 1.1 WHERE [CategoryId] = 5

// 여러 속성 동시 업데이트
await context.Products
    .Where(p => p.IsDiscontinued)
    .ExecuteUpdateAsync(setters => setters
        .SetProperty(p => p.Price, p => p.Price * 0.5m)
        .SetProperty(p => p.Status, ProductStatus.Clearance)
        .SetProperty(p => p.ModifiedAt, DateTime.UtcNow));
```

### 조건부 업데이트

```csharp
// 복잡한 조건
await context.Products
    .Where(p => p.Stock < 10 && p.IsActive)
    .ExecuteUpdateAsync(s => s
        .SetProperty(p => p.Status, ProductStatus.LowStock));

// 서브쿼리 사용
await context.Products
    .Where(p => context.Orders
        .Where(o => o.OrderDate > DateTime.UtcNow.AddMonths(-6))
        .SelectMany(o => o.OrderItems)
        .Any(oi => oi.ProductId == p.Id))
    .ExecuteUpdateAsync(s => s
        .SetProperty(p => p.IsPopular, true));
```

---

## 18.2 ExecuteDelete (EF Core 7+)

### 기본 사용법

```csharp
// 단순 삭제
await context.Products
    .Where(p => p.IsDiscontinued)
    .ExecuteDeleteAsync();

// SQL: DELETE FROM [Products] WHERE [IsDiscontinued] = 1

// 날짜 기반 삭제
await context.Logs
    .Where(l => l.CreatedAt < DateTime.UtcNow.AddYears(-1))
    .ExecuteDeleteAsync();

// 반환값: 삭제된 행 수
int deletedCount = await context.TempData
    .Where(t => t.ExpiresAt < DateTime.UtcNow)
    .ExecuteDeleteAsync();

Console.WriteLine($"{deletedCount}개 삭제됨");
```

### 관련 데이터 삭제

```csharp
// 주의: Cascade Delete는 DB 설정에 따름

// 먼저 자식 삭제
await context.OrderItems
    .Where(oi => context.Orders
        .Where(o => o.Status == OrderStatus.Cancelled)
        .Select(o => o.Id)
        .Contains(oi.OrderId))
    .ExecuteDeleteAsync();

// 그 다음 부모 삭제
await context.Orders
    .Where(o => o.Status == OrderStatus.Cancelled)
    .ExecuteDeleteAsync();
```

---

## 주의사항

### ChangeTracker와 동기화

```csharp
// ❌ 문제: 캐시된 데이터와 불일치
var product = await context.Products.FindAsync(1);
Console.WriteLine(product.Price);  // 100

// 다른 곳에서 ExecuteUpdate 실행
await context.Products
    .Where(p => p.Id == 1)
    .ExecuteUpdateAsync(s => s.SetProperty(p => p.Price, 150));

Console.WriteLine(product.Price);  // 여전히 100 (캐시된 값)

// ✅ 해결책: 캐시 초기화
context.ChangeTracker.Clear();

var product = await context.Products.FindAsync(1);
Console.WriteLine(product.Price);  // 150
```

### 필터와 인터셉터

```csharp
// ⚠️ 전역 쿼리 필터 적용됨
modelBuilder.Entity<Product>().HasQueryFilter(p => !p.IsDeleted);

await context.Products
    .Where(p => p.CategoryId == 5)
    .ExecuteDeleteAsync();
// 실제 SQL: DELETE ... WHERE CategoryId = 5 AND IsDeleted = 0

// 필터 무시
await context.Products
    .IgnoreQueryFilters()
    .Where(p => p.CategoryId == 5)
    .ExecuteDeleteAsync();

// ⚠️ SaveChanges 이벤트/인터셉터 실행 안 됨
// 감사 로그 등 수동 처리 필요
```

---

## 18.3 서드파티 라이브러리 활용

### EFCore.BulkExtensions

```csharp
// NuGet: EFCore.BulkExtensions

// 대량 삽입
var products = GenerateProducts(100000);
await context.BulkInsertAsync(products);

// 대량 업데이트
products.ForEach(p => p.Price *= 1.1m);
await context.BulkUpdateAsync(products);

// 대량 삭제
await context.BulkDeleteAsync(products);

// 옵션 설정
await context.BulkInsertAsync(products, config =>
{
    config.BatchSize = 5000;
    config.SetOutputIdentity = true;  // ID 반환
    config.UseTempDB = true;
});

// Upsert (Insert or Update)
await context.BulkInsertOrUpdateAsync(products);

// 조건부 업데이트
await context.BulkUpdateAsync(products,
    updateColumns: p => new { p.Price, p.ModifiedAt });
```

### Z.EntityFramework.Extensions

```csharp
// NuGet: Z.EntityFramework.Extensions.EFCore

// 대량 삽입
await context.BulkInsertAsync(products);

// 대량 업데이트 (조건부)
await context.Products
    .Where(p => p.CategoryId == 5)
    .UpdateFromQueryAsync(p => new Product
    {
        Price = p.Price * 1.1m
    });

// 대량 삭제
await context.Products
    .Where(p => p.IsDiscontinued)
    .DeleteFromQueryAsync();

// 배치 설정
context.BulkSaveChanges(options =>
{
    options.BatchSize = 1000;
    options.BatchTimeout = 180;
});
```

### 성능 비교

```
10만 건 삽입 성능 비교:

SaveChanges (개별)     : ~300초
SaveChanges (배치)     : ~60초
ExecuteUpdate          : N/A (삽입 불가)
BulkInsert            : ~3초
SqlBulkCopy           : ~2초
```

---

## 네이티브 벌크 작업

### SqlBulkCopy (SQL Server)

```csharp
using Microsoft.Data.SqlClient;

public async Task BulkInsertAsync(IEnumerable<Product> products)
{
    var dataTable = new DataTable();
    dataTable.Columns.Add("Name", typeof(string));
    dataTable.Columns.Add("Price", typeof(decimal));
    dataTable.Columns.Add("CategoryId", typeof(int));

    foreach (var product in products)
    {
        dataTable.Rows.Add(product.Name, product.Price, product.CategoryId);
    }

    using var connection = new SqlConnection(connectionString);
    await connection.OpenAsync();

    using var bulkCopy = new SqlBulkCopy(connection)
    {
        DestinationTableName = "Products",
        BatchSize = 10000
    };

    bulkCopy.ColumnMappings.Add("Name", "Name");
    bulkCopy.ColumnMappings.Add("Price", "Price");
    bulkCopy.ColumnMappings.Add("CategoryId", "CategoryId");

    await bulkCopy.WriteToServerAsync(dataTable);
}
```

### PostgreSQL COPY

```csharp
using Npgsql;

public async Task BulkInsertPostgresAsync(IEnumerable<Product> products)
{
    using var connection = new NpgsqlConnection(connectionString);
    await connection.OpenAsync();

    using var writer = await connection.BeginBinaryImportAsync(
        "COPY products (name, price, category_id) FROM STDIN (FORMAT BINARY)");

    foreach (var product in products)
    {
        await writer.StartRowAsync();
        await writer.WriteAsync(product.Name, NpgsqlDbType.Text);
        await writer.WriteAsync(product.Price, NpgsqlDbType.Numeric);
        await writer.WriteAsync(product.CategoryId, NpgsqlDbType.Integer);
    }

    await writer.CompleteAsync();
}
```

---

## 요약

| 방법 | 속도 | 복잡도 | 사용 시나리오 |
|------|------|--------|-------------|
| ExecuteUpdate/Delete | 빠름 | 낮음 | 조건부 대량 수정/삭제 |
| BulkExtensions | 매우 빠름 | 중간 | 엔티티 기반 대량 작업 |
| SqlBulkCopy | 가장 빠름 | 높음 | 초대량 삽입 |

## 다음 장 예고

다음 장에서는 연결 풀링과 DbContext 풀링을 통한 성능 최적화를 알아봅니다.
