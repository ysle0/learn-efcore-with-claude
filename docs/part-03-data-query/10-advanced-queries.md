# Chapter 10: 고급 쿼리 기법

## 개요

EF Core에서 복잡한 쿼리를 작성하는 방법을 알아봅니다. GroupBy, 조인, 서브쿼리, 원시 SQL 쿼리까지 다룹니다.

---

## 10.1 그룹화(GroupBy)와 집계 함수

### 간략 설명
GroupBy를 사용하여 데이터를 그룹화하고 집계 함수를 적용하는 방법을 알아봅니다.

### 기본 GroupBy

```csharp
// 카테고리별 제품 수
var productCounts = await context.Products
    .GroupBy(p => p.CategoryId)
    .Select(g => new
    {
        CategoryId = g.Key,
        Count = g.Count()
    })
    .ToListAsync();

// SQL: SELECT CategoryId, COUNT(*) FROM Products GROUP BY CategoryId

// 월별 주문 통계
var monthlyStats = await context.Orders
    .GroupBy(o => new { o.OrderDate.Year, o.OrderDate.Month })
    .Select(g => new
    {
        Year = g.Key.Year,
        Month = g.Key.Month,
        OrderCount = g.Count(),
        TotalAmount = g.Sum(o => o.TotalAmount),
        AverageAmount = g.Average(o => o.TotalAmount)
    })
    .OrderBy(x => x.Year)
    .ThenBy(x => x.Month)
    .ToListAsync();
```

### 복합 그룹화

```csharp
// 여러 컬럼으로 그룹화
var salesByRegionAndCategory = await context.Orders
    .GroupBy(o => new
    {
        o.Customer.Region,
        o.OrderItems.First().Product.CategoryId
    })
    .Select(g => new
    {
        Region = g.Key.Region,
        CategoryId = g.Key.CategoryId,
        TotalSales = g.Sum(o => o.TotalAmount),
        OrderCount = g.Count()
    })
    .ToListAsync();
```

### Having 절 (그룹 필터링)

```csharp
// 주문 5건 이상인 고객만
var frequentCustomers = await context.Orders
    .GroupBy(o => o.CustomerId)
    .Where(g => g.Count() >= 5)  // Having 절
    .Select(g => new
    {
        CustomerId = g.Key,
        OrderCount = g.Count(),
        TotalSpent = g.Sum(o => o.TotalAmount)
    })
    .ToListAsync();

// SQL:
// SELECT CustomerId, COUNT(*), SUM(TotalAmount)
// FROM Orders
// GROUP BY CustomerId
// HAVING COUNT(*) >= 5
```

### 집계 함수들

```csharp
var stats = await context.Products
    .GroupBy(p => p.CategoryId)
    .Select(g => new
    {
        CategoryId = g.Key,

        // 개수
        Count = g.Count(),
        CountWithCondition = g.Count(p => p.IsActive),

        // 합계
        TotalStock = g.Sum(p => p.StockQuantity),

        // 평균
        AvgPrice = g.Average(p => p.Price),

        // 최소/최대
        MinPrice = g.Min(p => p.Price),
        MaxPrice = g.Max(p => p.Price),

        // 첫 번째/마지막 (EF Core 6+)
        FirstProduct = g.OrderBy(p => p.Name).First().Name,
        LastProduct = g.OrderByDescending(p => p.Name).First().Name
    })
    .ToListAsync();
```

### 중첩 그룹화

```csharp
// 연도별, 월별 매출
var salesByYearMonth = await context.Orders
    .GroupBy(o => o.OrderDate.Year)
    .Select(yearGroup => new
    {
        Year = yearGroup.Key,
        TotalSales = yearGroup.Sum(o => o.TotalAmount),
        Months = yearGroup
            .GroupBy(o => o.OrderDate.Month)
            .Select(monthGroup => new
            {
                Month = monthGroup.Key,
                Sales = monthGroup.Sum(o => o.TotalAmount)
            })
            .ToList()
    })
    .ToListAsync();
```

---

## 10.2 조인(Join) 연산

### 간략 설명
여러 엔티티를 명시적으로 조인하는 방법을 알아봅니다. 네비게이션 속성이 없는 경우에 유용합니다.

### Inner Join

```csharp
// 메서드 구문
var results = await context.Orders
    .Join(
        context.Customers,
        order => order.CustomerId,
        customer => customer.Id,
        (order, customer) => new
        {
            OrderId = order.Id,
            OrderDate = order.OrderDate,
            CustomerName = customer.Name
        })
    .ToListAsync();

// 쿼리 구문
var results = await (
    from order in context.Orders
    join customer in context.Customers on order.CustomerId equals customer.Id
    select new
    {
        OrderId = order.Id,
        OrderDate = order.OrderDate,
        CustomerName = customer.Name
    })
    .ToListAsync();

// SQL: SELECT o.Id, o.OrderDate, c.Name FROM Orders o INNER JOIN Customers c ON o.CustomerId = c.Id
```

### Left Join

```csharp
// Left Outer Join
var results = await (
    from customer in context.Customers
    join order in context.Orders on customer.Id equals order.CustomerId into orders
    from order in orders.DefaultIfEmpty()  // Left Join
    select new
    {
        CustomerName = customer.Name,
        OrderId = order != null ? order.Id : (int?)null,
        OrderDate = order != null ? order.OrderDate : (DateTime?)null
    })
    .ToListAsync();

// 메서드 구문 (GroupJoin + SelectMany)
var results = await context.Customers
    .GroupJoin(
        context.Orders,
        customer => customer.Id,
        order => order.CustomerId,
        (customer, orders) => new { customer, orders })
    .SelectMany(
        x => x.orders.DefaultIfEmpty(),
        (x, order) => new
        {
            CustomerName = x.customer.Name,
            OrderId = order != null ? order.Id : (int?)null
        })
    .ToListAsync();
```

### 복합 키 조인

```csharp
// 여러 컬럼으로 조인
var results = await (
    from orderItem in context.OrderItems
    join priceHistory in context.PriceHistories
        on new { orderItem.ProductId, orderItem.OrderDate }
        equals new { priceHistory.ProductId, OrderDate = priceHistory.EffectiveDate }
    select new
    {
        orderItem.ProductId,
        orderItem.Quantity,
        HistoricalPrice = priceHistory.Price
    })
    .ToListAsync();
```

### 네비게이션 속성 vs 명시적 조인

```csharp
// ✅ 권장: 네비게이션 속성 사용
var ordersWithCustomer = await context.Orders
    .Include(o => o.Customer)
    .Select(o => new
    {
        o.Id,
        CustomerName = o.Customer.Name
    })
    .ToListAsync();

// 명시적 조인이 필요한 경우:
// 1. 네비게이션 속성이 없는 경우
// 2. 복잡한 조인 조건이 필요한 경우
// 3. 성능 최적화가 필요한 경우
```

### Cross Join

```csharp
// Cross Join (카테시안 곱)
var combinations = await (
    from product in context.Products
    from category in context.Categories
    select new
    {
        ProductName = product.Name,
        CategoryName = category.Name
    })
    .ToListAsync();

// 메서드 구문
var combinations = await context.Products
    .SelectMany(
        p => context.Categories,
        (product, category) => new
        {
            ProductName = product.Name,
            CategoryName = category.Name
        })
    .ToListAsync();
```

---

## 10.3 서브쿼리

### 간략 설명
LINQ에서 서브쿼리를 사용하여 복잡한 조건을 표현하는 방법을 알아봅니다.

### WHERE 절 서브쿼리

```csharp
// 주문이 있는 고객만 조회
var customersWithOrders = await context.Customers
    .Where(c => context.Orders.Any(o => o.CustomerId == c.Id))
    .ToListAsync();

// SQL: SELECT * FROM Customers c WHERE EXISTS (SELECT 1 FROM Orders o WHERE o.CustomerId = c.Id)

// 평균 이상 가격의 제품
var avgPrice = context.Products.Average(p => p.Price);
var expensiveProducts = await context.Products
    .Where(p => p.Price > avgPrice)
    .ToListAsync();

// 더 효율적인 방법 (단일 쿼리)
var expensiveProducts = await context.Products
    .Where(p => p.Price > context.Products.Average(p2 => p2.Price))
    .ToListAsync();
```

### SELECT 절 서브쿼리

```csharp
// 각 카테고리별 제품 수 포함
var categoriesWithCount = await context.Categories
    .Select(c => new
    {
        c.Id,
        c.Name,
        ProductCount = context.Products.Count(p => p.CategoryId == c.Id)
    })
    .ToListAsync();

// 최근 주문 날짜 포함
var customersWithLastOrder = await context.Customers
    .Select(c => new
    {
        c.Id,
        c.Name,
        LastOrderDate = context.Orders
            .Where(o => o.CustomerId == c.Id)
            .OrderByDescending(o => o.OrderDate)
            .Select(o => (DateTime?)o.OrderDate)
            .FirstOrDefault()
    })
    .ToListAsync();
```

### 상관 서브쿼리

```csharp
// 각 카테고리에서 가장 비싼 제품
var topProductsByCategory = await context.Products
    .Where(p => p.Price == context.Products
        .Where(p2 => p2.CategoryId == p.CategoryId)
        .Max(p2 => p2.Price))
    .ToListAsync();

// 주문 금액이 해당 고객 평균 이상인 주문
var aboveAverageOrders = await context.Orders
    .Where(o => o.TotalAmount > context.Orders
        .Where(o2 => o2.CustomerId == o.CustomerId)
        .Average(o2 => o2.TotalAmount))
    .ToListAsync();
```

### FROM 절 서브쿼리

```csharp
// 파생 테이블 사용
var orderSummary = await context.Orders
    .GroupBy(o => o.CustomerId)
    .Select(g => new
    {
        CustomerId = g.Key,
        TotalOrders = g.Count(),
        TotalSpent = g.Sum(o => o.TotalAmount)
    })
    .Where(s => s.TotalSpent > 10000)
    .Join(
        context.Customers,
        s => s.CustomerId,
        c => c.Id,
        (s, c) => new
        {
            c.Name,
            s.TotalOrders,
            s.TotalSpent
        })
    .ToListAsync();
```

---

## 10.4 원시 SQL 쿼리 (FromSqlRaw, FromSqlInterpolated)

### 간략 설명
LINQ로 표현하기 어려운 복잡한 쿼리는 원시 SQL을 사용할 수 있습니다.

### FromSqlRaw

```csharp
// 기본 사용법
var products = await context.Products
    .FromSqlRaw("SELECT * FROM Products WHERE Price > 100")
    .ToListAsync();

// 매개변수 사용 (SQL Injection 방지)
var minPrice = 100;
var products = await context.Products
    .FromSqlRaw("SELECT * FROM Products WHERE Price > {0}", minPrice)
    .ToListAsync();

// 여러 매개변수
var products = await context.Products
    .FromSqlRaw(
        "SELECT * FROM Products WHERE Price > {0} AND CategoryId = {1}",
        minPrice, categoryId)
    .ToListAsync();
```

### FromSqlInterpolated (권장)

```csharp
// 문자열 보간 사용 (더 안전하고 가독성 좋음)
var minPrice = 100;
var products = await context.Products
    .FromSqlInterpolated($"SELECT * FROM Products WHERE Price > {minPrice}")
    .ToListAsync();

// 복잡한 쿼리
var searchTerm = "phone";
var categoryId = 5;
var products = await context.Products
    .FromSqlInterpolated($@"
        SELECT *
        FROM Products
        WHERE CategoryId = {categoryId}
          AND Name LIKE '%' + {searchTerm} + '%'
        ORDER BY Price DESC")
    .ToListAsync();
```

### LINQ와 조합

```csharp
// FromSql 후 LINQ 연산 추가 가능
var products = await context.Products
    .FromSqlInterpolated($"SELECT * FROM Products WHERE IsActive = 1")
    .Where(p => p.Price > 50)  // 추가 필터링
    .OrderBy(p => p.Name)       // 정렬
    .Include(p => p.Category)   // Include도 가능
    .ToListAsync();

// SQL: SELECT * FROM (SELECT * FROM Products WHERE IsActive = 1) AS p
//      WHERE p.Price > 50 ORDER BY p.Name
```

### 저장 프로시저 호출

```csharp
// 저장 프로시저 호출
var products = await context.Products
    .FromSqlRaw("EXEC GetProductsByCategory @CategoryId = {0}", categoryId)
    .ToListAsync();

// 출력 매개변수 (ADO.NET 직접 사용)
var outputParam = new SqlParameter
{
    ParameterName = "@TotalCount",
    SqlDbType = SqlDbType.Int,
    Direction = ParameterDirection.Output
};

await context.Database.ExecuteSqlRawAsync(
    "EXEC GetProductsWithCount @CategoryId, @TotalCount OUTPUT",
    new SqlParameter("@CategoryId", categoryId),
    outputParam);

var totalCount = (int)outputParam.Value;
```

### 비엔티티 쿼리 (EF Core 8+)

```csharp
// 엔티티가 아닌 타입으로 쿼리
var stats = await context.Database
    .SqlQuery<ProductStats>($@"
        SELECT
            CategoryId,
            COUNT(*) AS ProductCount,
            AVG(Price) AS AveragePrice,
            SUM(StockQuantity) AS TotalStock
        FROM Products
        GROUP BY CategoryId")
    .ToListAsync();

public class ProductStats
{
    public int CategoryId { get; set; }
    public int ProductCount { get; set; }
    public decimal AveragePrice { get; set; }
    public int TotalStock { get; set; }
}
```

### ExecuteSql (DML)

```csharp
// UPDATE
var rowsAffected = await context.Database
    .ExecuteSqlInterpolatedAsync($@"
        UPDATE Products
        SET Price = Price * 1.1
        WHERE CategoryId = {categoryId}");

// DELETE
await context.Database
    .ExecuteSqlInterpolatedAsync($@"
        DELETE FROM OrderItems
        WHERE OrderId IN (
            SELECT Id FROM Orders WHERE OrderDate < {cutoffDate}
        )");

// INSERT
await context.Database
    .ExecuteSqlInterpolatedAsync($@"
        INSERT INTO AuditLog (Action, Timestamp, UserId)
        VALUES ({action}, {DateTime.UtcNow}, {userId})");
```

### 원시 SQL 주의사항

```csharp
// ❌ SQL Injection 위험!
var userInput = "test'; DROP TABLE Products; --";
var products = await context.Products
    .FromSqlRaw($"SELECT * FROM Products WHERE Name = '{userInput}'")
    .ToListAsync();

// ✅ 안전한 방법
var products = await context.Products
    .FromSqlInterpolated($"SELECT * FROM Products WHERE Name = {userInput}")
    .ToListAsync();

// 또는
var products = await context.Products
    .FromSqlRaw("SELECT * FROM Products WHERE Name = {0}", userInput)
    .ToListAsync();
```

---

## EF.Functions - 데이터베이스 함수

### 문자열 함수

```csharp
// LIKE
var products = await context.Products
    .Where(p => EF.Functions.Like(p.Name, "%phone%"))
    .ToListAsync();

// 대소문자 구분 없는 Like (SQL Server Collation)
var products = await context.Products
    .Where(p => EF.Functions.Collate(p.Name, "Latin1_General_CI_AI")
        .Contains("PHONE"))
    .ToListAsync();
```

### 날짜 함수

```csharp
// DateDiff
var recentOrders = await context.Orders
    .Where(o => EF.Functions.DateDiffDay(o.OrderDate, DateTime.UtcNow) <= 30)
    .ToListAsync();

// DatePart (EF Core 6+)
var ordersInJanuary = await context.Orders
    .Where(o => o.OrderDate.Month == 1)
    .ToListAsync();
```

### 전체 텍스트 검색 (SQL Server)

```csharp
// Contains
var products = await context.Products
    .Where(p => EF.Functions.Contains(p.Description, "wireless AND bluetooth"))
    .ToListAsync();

// FreeText
var products = await context.Products
    .Where(p => EF.Functions.FreeText(p.Description, "wireless bluetooth headphones"))
    .ToListAsync();
```

---

## 요약

| 기법 | 사용 시나리오 | 주의사항 |
|------|-------------|---------|
| GroupBy | 집계, 통계 | 클라이언트 평가 주의 |
| Join | 명시적 조인 필요 시 | 네비게이션 우선 |
| 서브쿼리 | 복잡한 조건 | 성능 확인 필요 |
| FromSql | LINQ 한계 시 | SQL Injection 주의 |
| EF.Functions | DB 함수 사용 | 프로바이더별 차이 |

## 다음 장 예고

다음 장에서는 쿼리 필터와 전역 필터를 사용하여 Soft Delete, Multi-Tenancy를 구현하는 방법을 알아봅니다.
