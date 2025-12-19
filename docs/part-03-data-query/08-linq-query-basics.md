# Chapter 08: LINQ 쿼리 기초

## 개요

EF Core에서 LINQ를 사용하여 데이터를 조회하는 방법을 알아봅니다. 기본 쿼리 작성부터 필터링, 정렬, 페이징까지 다룹니다.

---

## 8.1 기본 쿼리 작성

### 간략 설명
EF Core는 LINQ(Language Integrated Query)를 사용하여 타입 안전한 쿼리를 작성할 수 있습니다.

### LINQ 쿼리 구문 vs 메서드 구문

```csharp
// 쿼리 구문 (Query Syntax)
var activeUsers = from u in context.Users
                  where u.IsActive
                  orderby u.Name
                  select u;

// 메서드 구문 (Method Syntax) - 권장
var activeUsers = context.Users
    .Where(u => u.IsActive)
    .OrderBy(u => u.Name);

// 둘 다 동일한 SQL 생성
// SELECT * FROM Users WHERE IsActive = 1 ORDER BY Name
```

### 쿼리 실행 시점

```csharp
// 1. 지연 실행 (Deferred Execution)
var query = context.Users.Where(u => u.IsActive);  // SQL 미실행
// 이 시점에서 query는 IQueryable - 아직 DB 접근 안 함

// 2. 즉시 실행 (Immediate Execution)
var users = await query.ToListAsync();     // SQL 실행
var count = await query.CountAsync();      // SQL 실행
var first = await query.FirstAsync();      // SQL 실행
var exists = await query.AnyAsync();       // SQL 실행

// 즉시 실행을 유발하는 메서드들
// ToList(), ToArray(), ToDictionary()
// First(), FirstOrDefault(), Single(), SingleOrDefault()
// Count(), Sum(), Average(), Min(), Max()
// Any(), All()
```

### 동기 vs 비동기

```csharp
// ❌ 동기 - 스레드 블로킹
var users = context.Users.ToList();

// ✅ 비동기 - 권장
var users = await context.Users.ToListAsync();

// 비동기 메서드들
await context.Users.ToListAsync();
await context.Users.FirstOrDefaultAsync(u => u.Id == 1);
await context.Users.CountAsync();
await context.Users.AnyAsync(u => u.IsActive);
await context.Users.FindAsync(1);  // 기본 키로 조회
```

---

## 8.2 Where, Select, OrderBy

### Where - 필터링

```csharp
// 단일 조건
var activeUsers = await context.Users
    .Where(u => u.IsActive)
    .ToListAsync();

// 복합 조건
var filteredUsers = await context.Users
    .Where(u => u.IsActive && u.Age >= 18)
    .ToListAsync();

// OR 조건
var users = await context.Users
    .Where(u => u.Role == "Admin" || u.Role == "Manager")
    .ToListAsync();

// 문자열 검색
var searchResults = await context.Users
    .Where(u => u.Name.Contains("Kim"))
    .ToListAsync();
// SQL: WHERE Name LIKE '%Kim%'

// 시작/끝 검색
var users1 = await context.Users
    .Where(u => u.Email.StartsWith("admin"))
    .ToListAsync();
// SQL: WHERE Email LIKE 'admin%'

var users2 = await context.Users
    .Where(u => u.Email.EndsWith("@company.com"))
    .ToListAsync();
// SQL: WHERE Email LIKE '%@company.com'

// NULL 체크
var usersWithBio = await context.Users
    .Where(u => u.Bio != null)
    .ToListAsync();

// 컬렉션 포함 여부
var roles = new[] { "Admin", "Manager", "Editor" };
var users = await context.Users
    .Where(u => roles.Contains(u.Role))
    .ToListAsync();
// SQL: WHERE Role IN ('Admin', 'Manager', 'Editor')
```

### Select - 프로젝션

```csharp
// 특정 속성만 선택
var names = await context.Users
    .Select(u => u.Name)
    .ToListAsync();
// SQL: SELECT Name FROM Users

// 익명 타입으로 프로젝션
var userDtos = await context.Users
    .Select(u => new { u.Id, u.Name, u.Email })
    .ToListAsync();

// DTO로 프로젝션
public class UserDto
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public string Email { get; set; } = string.Empty;
}

var userDtos = await context.Users
    .Select(u => new UserDto
    {
        Id = u.Id,
        Name = u.Name,
        Email = u.Email
    })
    .ToListAsync();

// 계산된 값
var userSummaries = await context.Users
    .Select(u => new
    {
        u.Name,
        FullName = u.FirstName + " " + u.LastName,
        OrderCount = u.Orders.Count,
        TotalSpent = u.Orders.Sum(o => o.TotalAmount)
    })
    .ToListAsync();
```

### OrderBy - 정렬

```csharp
// 오름차순
var users = await context.Users
    .OrderBy(u => u.Name)
    .ToListAsync();
// SQL: ORDER BY Name ASC

// 내림차순
var users = await context.Users
    .OrderByDescending(u => u.CreatedAt)
    .ToListAsync();
// SQL: ORDER BY CreatedAt DESC

// 다중 정렬
var users = await context.Users
    .OrderBy(u => u.LastName)
    .ThenBy(u => u.FirstName)
    .ToListAsync();
// SQL: ORDER BY LastName ASC, FirstName ASC

var users = await context.Users
    .OrderByDescending(u => u.IsActive)
    .ThenBy(u => u.Name)
    .ToListAsync();
// SQL: ORDER BY IsActive DESC, Name ASC
```

---

## 8.3 First, Single, Find 차이점

### 메서드 비교

| 메서드 | 결과 없음 | 결과 1개 | 결과 여러 개 |
|--------|----------|---------|------------|
| First | 예외 | 반환 | 첫 번째 반환 |
| FirstOrDefault | null/default | 반환 | 첫 번째 반환 |
| Single | 예외 | 반환 | 예외 |
| SingleOrDefault | null/default | 반환 | 예외 |
| Find | null | 반환 | - (PK 검색) |

### First / FirstOrDefault

```csharp
// First - 결과 없으면 예외
try
{
    var user = await context.Users
        .Where(u => u.Email == "admin@example.com")
        .FirstAsync();
}
catch (InvalidOperationException)
{
    // 결과가 없는 경우
}

// FirstOrDefault - 결과 없으면 null (권장)
var user = await context.Users
    .Where(u => u.Email == "admin@example.com")
    .FirstOrDefaultAsync();

if (user == null)
{
    // 결과가 없는 경우 처리
}

// 조건 직접 전달
var user = await context.Users
    .FirstOrDefaultAsync(u => u.Email == "admin@example.com");

// 정렬 후 첫 번째 (가장 최근)
var latestOrder = await context.Orders
    .OrderByDescending(o => o.CreatedAt)
    .FirstOrDefaultAsync();
```

### Single / SingleOrDefault

```csharp
// Single - 정확히 1개가 아니면 예외
var user = await context.Users
    .Where(u => u.Id == 1)
    .SingleAsync();  // 없거나 2개 이상이면 예외

// SingleOrDefault - 1개 또는 없음 (2개 이상은 예외)
var user = await context.Users
    .SingleOrDefaultAsync(u => u.Email == "unique@example.com");

// 고유성이 보장되는 경우에만 사용
var config = await context.AppConfigs
    .SingleOrDefaultAsync(c => c.Key == "SiteName");
```

### Find - 기본 키 검색

```csharp
// Find - DbSet에서만 사용, 기본 키로 검색
var user = await context.Users.FindAsync(1);

// 복합 키
var orderItem = await context.OrderItems.FindAsync(orderId, productId);

// Find의 특별한 점:
// 1. 먼저 로컬 캐시(Change Tracker) 검색
// 2. 없으면 DB 쿼리
// 3. AsNoTracking()과 함께 사용 불가

// Find vs FirstOrDefault
var user1 = await context.Users.FindAsync(1);
// 캐시 먼저 확인 → DB 쿼리 (필요시)

var user2 = await context.Users.FirstOrDefaultAsync(u => u.Id == 1);
// 항상 DB 쿼리
```

---

## 8.4 Any, All, Count 메서드

### Any - 존재 여부

```csharp
// 조건에 맞는 항목이 하나라도 있는지
bool hasActiveUsers = await context.Users
    .AnyAsync(u => u.IsActive);
// SQL: SELECT CASE WHEN EXISTS(SELECT 1 FROM Users WHERE IsActive = 1) THEN 1 ELSE 0 END

// 테이블에 데이터가 있는지
bool hasAnyUsers = await context.Users.AnyAsync();

// 조건부 로직
if (await context.Users.AnyAsync(u => u.Email == email))
{
    throw new Exception("이미 사용 중인 이메일입니다.");
}

// 관련 데이터 존재 여부
var blogsWithPosts = await context.Blogs
    .Where(b => b.Posts.Any())
    .ToListAsync();

var blogsWithRecentPosts = await context.Blogs
    .Where(b => b.Posts.Any(p => p.PublishedAt > DateTime.UtcNow.AddDays(-7)))
    .ToListAsync();
```

### All - 모든 항목 조건 충족

```csharp
// 모든 항목이 조건을 만족하는지
bool allUsersActive = await context.Users
    .AllAsync(u => u.IsActive);
// SQL: SELECT CASE WHEN NOT EXISTS(SELECT 1 FROM Users WHERE IsActive = 0) THEN 1 ELSE 0 END

// 모든 주문이 배송되었는지
bool allShipped = await context.Orders
    .Where(o => o.CustomerId == customerId)
    .AllAsync(o => o.Status == OrderStatus.Shipped);

// 주의: 빈 컬렉션은 All이 true 반환
var hasOrders = await context.Orders.AnyAsync(o => o.CustomerId == 999);  // false
var allShipped = await context.Orders
    .Where(o => o.CustomerId == 999)
    .AllAsync(o => o.Status == OrderStatus.Shipped);  // true (빈 컬렉션)
```

### Count - 개수 세기

```csharp
// 전체 개수
int totalUsers = await context.Users.CountAsync();

// 조건부 개수
int activeUsers = await context.Users.CountAsync(u => u.IsActive);

// LongCount - 대용량
long totalRecords = await context.Logs.LongCountAsync();

// 관련 데이터 개수
var blogStats = await context.Blogs
    .Select(b => new
    {
        b.Name,
        PostCount = b.Posts.Count,
        PublishedPostCount = b.Posts.Count(p => p.IsPublished)
    })
    .ToListAsync();

// 페이징에서 사용
var pageSize = 10;
var totalCount = await context.Products.CountAsync();
var totalPages = (int)Math.Ceiling(totalCount / (double)pageSize);
```

### Sum, Average, Min, Max

```csharp
// 합계
decimal totalSales = await context.Orders
    .SumAsync(o => o.TotalAmount);

decimal? totalSales = await context.Orders
    .Where(o => o.Status == OrderStatus.Completed)
    .SumAsync(o => (decimal?)o.TotalAmount);  // nullable로 빈 결과 처리

// 평균
double avgOrderAmount = await context.Orders
    .AverageAsync(o => o.TotalAmount);

// 최소/최대
DateTime? firstOrderDate = await context.Orders
    .MinAsync(o => (DateTime?)o.OrderDate);

decimal maxOrderAmount = await context.Orders
    .MaxAsync(o => o.TotalAmount);

// 복합 집계
var orderStats = await context.Orders
    .GroupBy(o => 1)  // 전체를 하나의 그룹으로
    .Select(g => new
    {
        Count = g.Count(),
        TotalAmount = g.Sum(o => o.TotalAmount),
        AverageAmount = g.Average(o => o.TotalAmount),
        MinAmount = g.Min(o => o.TotalAmount),
        MaxAmount = g.Max(o => o.TotalAmount)
    })
    .FirstOrDefaultAsync();
```

---

## 페이징

### Skip과 Take

```csharp
// 기본 페이징
int page = 1;
int pageSize = 10;

var products = await context.Products
    .OrderBy(p => p.Name)
    .Skip((page - 1) * pageSize)
    .Take(pageSize)
    .ToListAsync();

// SQL: SELECT * FROM Products ORDER BY Name OFFSET 0 ROWS FETCH NEXT 10 ROWS ONLY

// 페이징 결과 클래스
public class PagedResult<T>
{
    public List<T> Items { get; set; } = new();
    public int TotalCount { get; set; }
    public int Page { get; set; }
    public int PageSize { get; set; }
    public int TotalPages => (int)Math.Ceiling(TotalCount / (double)PageSize);
    public bool HasPreviousPage => Page > 1;
    public bool HasNextPage => Page < TotalPages;
}

// 페이징 헬퍼 메서드
public static async Task<PagedResult<T>> ToPagedResultAsync<T>(
    this IQueryable<T> query,
    int page,
    int pageSize)
{
    var totalCount = await query.CountAsync();

    var items = await query
        .Skip((page - 1) * pageSize)
        .Take(pageSize)
        .ToListAsync();

    return new PagedResult<T>
    {
        Items = items,
        TotalCount = totalCount,
        Page = page,
        PageSize = pageSize
    };
}

// 사용
var result = await context.Products
    .Where(p => p.IsActive)
    .OrderBy(p => p.Name)
    .ToPagedResultAsync(page: 2, pageSize: 20);
```

### 키셋 페이징 (성능 최적화)

```csharp
// 오프셋 페이징의 문제: 큰 오프셋에서 느림
// SELECT * FROM Products ORDER BY Name OFFSET 100000 ROWS FETCH NEXT 10 ROWS ONLY
// → 100,000개를 건너뛰어야 함

// 키셋 페이징 (Cursor-based)
var products = await context.Products
    .Where(p => p.Id > lastSeenId)  // 마지막으로 본 ID 이후
    .OrderBy(p => p.Id)
    .Take(pageSize)
    .ToListAsync();

// SQL: SELECT TOP 10 * FROM Products WHERE Id > @lastSeenId ORDER BY Id
// → 인덱스 활용, 매우 빠름

// 다중 컬럼 정렬 시
var products = await context.Products
    .Where(p => p.Name.CompareTo(lastName) > 0 ||
               (p.Name == lastName && p.Id > lastId))
    .OrderBy(p => p.Name)
    .ThenBy(p => p.Id)
    .Take(pageSize)
    .ToListAsync();
```

---

## 쿼리 디버깅

### 생성된 SQL 확인

```csharp
// 방법 1: ToQueryString()
var query = context.Users
    .Where(u => u.IsActive)
    .OrderBy(u => u.Name);

string sql = query.ToQueryString();
Console.WriteLine(sql);

// 방법 2: 로깅 설정
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
{
    optionsBuilder
        .UseSqlServer(connectionString)
        .LogTo(Console.WriteLine, LogLevel.Information)
        .EnableSensitiveDataLogging();  // 개발 환경에서만!
}

// 방법 3: DbContext 이벤트
public class MyDbContext : DbContext
{
    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        optionsBuilder.LogTo(
            message => Debug.WriteLine(message),
            new[] { DbLoggerCategory.Database.Command.Name },
            LogLevel.Information);
    }
}
```

---

## 요약

| 메서드 | 용도 | SQL 생성 |
|--------|------|---------|
| Where | 필터링 | WHERE |
| Select | 프로젝션 | SELECT |
| OrderBy/ThenBy | 정렬 | ORDER BY |
| First/Single | 단일 조회 | TOP 1 / TOP 2 |
| Find | PK 조회 | WHERE PK = |
| Any/All | 존재/조건 확인 | EXISTS |
| Count | 개수 | COUNT |
| Skip/Take | 페이징 | OFFSET/FETCH |

## 다음 장 예고

다음 장에서는 관련 데이터를 로딩하는 세 가지 전략(즉시, 지연, 명시적)을 알아봅니다.
