# Chapter 18: 쿼리 성능 최적화

## 개요

EF Core 쿼리 성능을 최적화하는 방법을 알아봅니다. 생성된 SQL 분석, N+1 문제 해결, 프로젝션, 컴파일된 쿼리를 다룹니다.

---

## 17.1 생성된 SQL 확인하기

### 로깅 설정

```csharp
// DbContext 설정
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
{
    optionsBuilder
        .UseSqlServer(connectionString)
        .LogTo(Console.WriteLine, LogLevel.Information)
        .EnableSensitiveDataLogging()  // 매개변수 값 표시 (개발 환경만!)
        .EnableDetailedErrors();
}

// 또는 DI 설정
services.AddDbContext<AppDbContext>((sp, options) =>
{
    options.UseSqlServer(connectionString);

    if (env.IsDevelopment())
    {
        options.LogTo(Console.WriteLine, LogLevel.Information);
        options.EnableSensitiveDataLogging();
    }
});
```

### ToQueryString()

```csharp
// 쿼리 문자열 직접 확인
var query = context.Products
    .Where(p => p.Price > 100)
    .OrderBy(p => p.Name);

string sql = query.ToQueryString();
Console.WriteLine(sql);

// 출력:
// SELECT [p].[Id], [p].[Name], [p].[Price]
// FROM [Products] AS [p]
// WHERE [p].[Price] > @__price_0
// ORDER BY [p].[Name]
```

### 쿼리 태그

```csharp
// 쿼리에 주석 추가 (디버깅/모니터링용)
var products = await context.Products
    .TagWith("GetActiveProducts - HomePage")
    .Where(p => p.IsActive)
    .ToListAsync();

// 생성되는 SQL:
// -- GetActiveProducts - HomePage
// SELECT * FROM Products WHERE IsActive = 1
```

---

## 17.2 N+1 문제 해결

### N+1 문제 예시

```csharp
// ❌ N+1 문제 발생
var blogs = await context.Blogs.ToListAsync();  // 1번 쿼리

foreach (var blog in blogs)
{
    Console.WriteLine($"Blog: {blog.Name}");
    foreach (var post in blog.Posts)  // N번 추가 쿼리!
    {
        Console.WriteLine($"  Post: {post.Title}");
    }
}

// 블로그 100개 → 101번의 쿼리 실행!
```

### 해결책 1: Include

```csharp
// ✅ 즉시 로딩으로 해결
var blogs = await context.Blogs
    .Include(b => b.Posts)
    .ToListAsync();  // 1번 쿼리 (JOIN)

foreach (var blog in blogs)
{
    foreach (var post in blog.Posts)  // 추가 쿼리 없음
    {
        Console.WriteLine(post.Title);
    }
}
```

### 해결책 2: 분할 쿼리

```csharp
// 여러 컬렉션 Include 시 카테시안 곱 방지
var blogs = await context.Blogs
    .Include(b => b.Posts)
    .Include(b => b.Tags)
    .AsSplitQuery()  // 별도 쿼리로 분리
    .ToListAsync();

// 3번의 쿼리로 분리:
// SELECT * FROM Blogs
// SELECT * FROM Posts WHERE BlogId IN (...)
// SELECT * FROM Tags WHERE BlogId IN (...)
```

### 해결책 3: 프로젝션

```csharp
// 필요한 데이터만 선택
var blogSummaries = await context.Blogs
    .Select(b => new BlogSummary
    {
        Name = b.Name,
        PostCount = b.Posts.Count,
        LatestPostTitle = b.Posts
            .OrderByDescending(p => p.CreatedAt)
            .Select(p => p.Title)
            .FirstOrDefault()
    })
    .ToListAsync();  // 1번 쿼리
```

---

## 17.3 프로젝션(Projection) 활용

### Select를 사용한 최적화

```csharp
// ❌ 전체 엔티티 로드
var products = await context.Products.ToListAsync();
var names = products.Select(p => p.Name);

// ✅ 필요한 컬럼만 조회
var names = await context.Products
    .Select(p => p.Name)
    .ToListAsync();

// SQL: SELECT [p].[Name] FROM [Products] AS [p]
```

### DTO 프로젝션

```csharp
public class ProductDto
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public decimal Price { get; set; }
    public string CategoryName { get; set; } = string.Empty;
}

var products = await context.Products
    .Select(p => new ProductDto
    {
        Id = p.Id,
        Name = p.Name,
        Price = p.Price,
        CategoryName = p.Category.Name  // JOIN 발생
    })
    .ToListAsync();

// 필요한 컬럼만 SELECT + 자동 JOIN
```

### 집계와 프로젝션

```csharp
var stats = await context.Products
    .GroupBy(p => p.CategoryId)
    .Select(g => new
    {
        CategoryId = g.Key,
        Count = g.Count(),
        AvgPrice = g.Average(p => p.Price),
        MaxPrice = g.Max(p => p.Price)
    })
    .ToListAsync();
```

---

## 17.4 컴파일된 쿼리(Compiled Queries)

### 간략 설명
자주 사용되는 쿼리를 미리 컴파일하여 실행 시간을 단축합니다.

### 기본 사용법

```csharp
public class ProductQueries
{
    // 컴파일된 쿼리 정의
    private static readonly Func<AppDbContext, int, Task<Product?>> GetProductById =
        EF.CompileAsyncQuery(
            (AppDbContext context, int id) =>
                context.Products.FirstOrDefault(p => p.Id == id));

    private static readonly Func<AppDbContext, decimal, IAsyncEnumerable<Product>> GetExpensiveProducts =
        EF.CompileAsyncQuery(
            (AppDbContext context, decimal minPrice) =>
                context.Products.Where(p => p.Price > minPrice));

    // 사용
    public async Task<Product?> FindProductAsync(AppDbContext context, int id)
    {
        return await GetProductById(context, id);
    }

    public async Task<List<Product>> GetExpensiveAsync(AppDbContext context, decimal minPrice)
    {
        var result = new List<Product>();
        await foreach (var product in GetExpensiveProducts(context, minPrice))
        {
            result.Add(product);
        }
        return result;
    }
}
```

### 컴파일된 쿼리 장점

```
일반 쿼리 실행 과정:
1. LINQ 표현식 분석
2. 쿼리 캐시 확인
3. 캐시 미스 시 SQL 생성
4. SQL 실행

컴파일된 쿼리:
1. SQL 즉시 실행 (사전 컴파일됨)

성능 향상: 반복 실행 시 10-20% 개선
```

### 제한사항

```csharp
// ❌ 지원되지 않는 패턴

// Include는 직접 사용 불가
var query = EF.CompileQuery((AppDbContext ctx) =>
    ctx.Blogs.Include(b => b.Posts));  // 컴파일 오류

// 동적 정렬 불가
var query = EF.CompileQuery((AppDbContext ctx, string orderBy) =>
    ctx.Products.OrderBy(p => orderBy));  // 작동 안 함

// ✅ 해결책: 별도 컴파일된 쿼리 정의
private static readonly Func<AppDbContext, IAsyncEnumerable<Product>> GetByNameAsc =
    EF.CompileAsyncQuery((AppDbContext ctx) =>
        ctx.Products.OrderBy(p => p.Name));

private static readonly Func<AppDbContext, IAsyncEnumerable<Product>> GetByPriceDesc =
    EF.CompileAsyncQuery((AppDbContext ctx) =>
        ctx.Products.OrderByDescending(p => p.Price));
```

---

## 추가 최적화 팁

### 불필요한 추적 비활성화

```csharp
// 읽기 전용 쿼리
var products = await context.Products
    .AsNoTracking()
    .Where(p => p.IsActive)
    .ToListAsync();

// 전역 설정
public class ReadOnlyContext : DbContext
{
    public ReadOnlyContext(DbContextOptions options) : base(options)
    {
        ChangeTracker.QueryTrackingBehavior = QueryTrackingBehavior.NoTracking;
    }
}
```

### 페이징 최적화

```csharp
// ❌ 큰 오프셋에서 느림
var page100 = await context.Products
    .OrderBy(p => p.Id)
    .Skip(99 * 20)  // 1980개 건너뛰기 (느림)
    .Take(20)
    .ToListAsync();

// ✅ 키셋 페이징
var page100 = await context.Products
    .Where(p => p.Id > lastSeenId)
    .OrderBy(p => p.Id)
    .Take(20)
    .ToListAsync();
```

### 배치 조회

```csharp
// ❌ 여러 번 조회
foreach (var id in productIds)
{
    var product = await context.Products.FindAsync(id);
}

// ✅ 한 번에 조회
var products = await context.Products
    .Where(p => productIds.Contains(p.Id))
    .ToDictionaryAsync(p => p.Id);
```

---

## 요약

| 기법 | 효과 | 사용 시나리오 |
|------|------|-------------|
| Include | N+1 해결 | 관련 데이터 필요 시 |
| AsSplitQuery | 카테시안 곱 방지 | 다중 컬렉션 |
| Select 프로젝션 | 데이터 전송 감소 | DTO 변환 |
| 컴파일된 쿼리 | 파싱 오버헤드 제거 | 반복 실행 쿼리 |
| AsNoTracking | 메모리/CPU 절약 | 읽기 전용 |

## 다음 장 예고

다음 장에서는 ExecuteUpdate/Delete와 벌크 작업 최적화를 알아봅니다.
