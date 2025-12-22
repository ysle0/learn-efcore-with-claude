# Chapter 06: N+1 문제 이해와 해결

## 개요

N+1 문제는 ORM을 사용할 때 가장 흔히 발생하는 성능 문제입니다. 이 문서에서는 N+1 문제가 무엇인지, 왜 발생하는지, 그리고 어떻게 해결하는지 상세히 알아봅니다.

---

## N+1 문제란?

### 간략 설명

**N+1 문제**는 1개의 쿼리로 N개의 데이터를 가져온 후, 각 데이터의 관련 엔티티를 조회하기 위해 N개의 추가 쿼리가 실행되는 현상입니다.

```
총 쿼리 수 = 1 (메인) + N (관련 데이터) = N+1
```

### 시각적 이해

```
┌─────────────────────────────────────────────────────────────┐
│                    N+1 문제 발생 과정                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1️⃣ 첫 번째 쿼리: 블로그 목록 조회                           │
│     SELECT * FROM Blogs                                     │
│     → 결과: Blog 1, Blog 2, Blog 3 ... Blog N               │
│                                                             │
│  2️⃣ 각 블로그마다 추가 쿼리 발생 (N번)                       │
│     SELECT * FROM Posts WHERE BlogId = 1  ─┐               │
│     SELECT * FROM Posts WHERE BlogId = 2   │               │
│     SELECT * FROM Posts WHERE BlogId = 3   ├─ N개의 쿼리    │
│     ...                                    │               │
│     SELECT * FROM Posts WHERE BlogId = N  ─┘               │
│                                                             │
│  총 쿼리 수: 1 + N = N+1 개                                  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 문제 발생 예시

### 엔티티 구조

```csharp
public class Blog
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public string Url { get; set; } = string.Empty;

    // Navigation Property
    public ICollection<Post> Posts { get; set; } = new List<Post>();
}

public class Post
{
    public int Id { get; set; }
    public string Title { get; set; } = string.Empty;
    public string Content { get; set; } = string.Empty;

    // FK Property (항상 로드됨)
    public int BlogId { get; set; }

    // Navigation Property (Include 필요)
    public Blog Blog { get; set; } = null!;
}
```

### N+1 문제 발생 코드

```csharp
// ❌ N+1 문제 발생!
var blogs = await context.Blogs.ToListAsync();  // 쿼리 1번

foreach (var blog in blogs)
{
    Console.WriteLine($"Blog: {blog.Name}");

    // 각 반복마다 추가 쿼리 발생!
    foreach (var post in blog.Posts)  // 쿼리 N번
    {
        Console.WriteLine($"  - {post.Title}");
    }
}
```

### 실행되는 SQL

```sql
-- 쿼리 1: 블로그 목록
SELECT [b].[Id], [b].[Name], [b].[Url]
FROM [Blogs] AS [b]

-- 쿼리 2: 첫 번째 블로그의 포스트
SELECT [p].[Id], [p].[Title], [p].[Content], [p].[BlogId]
FROM [Posts] AS [p]
WHERE [p].[BlogId] = 1

-- 쿼리 3: 두 번째 블로그의 포스트
SELECT [p].[Id], [p].[Title], [p].[Content], [p].[BlogId]
FROM [Posts] AS [p]
WHERE [p].[BlogId] = 2

-- ... N번 반복
```

### 성능 영향

| 블로그 수 | 쿼리 수 | 예상 소요 시간 (원격 DB) |
|----------|--------|------------------------|
| 10개 | 11 | ~110ms |
| 100개 | 101 | ~1초 |
| 1,000개 | 1,001 | ~10초 |
| 10,000개 | 10,001 | ~100초 |

---

## FK Property vs Navigation Property

### 핵심 차이점

```csharp
public class Post
{
    public int Id { get; set; }
    public string Title { get; set; }

    public int BlogId { get; set; }     // FK Property: 항상 로드됨
    public Blog Blog { get; set; }       // Navigation: Include 필요
}
```

```csharp
// Include 없이 조회
var post = await context.Posts.FirstAsync();

post.BlogId;  // ✅ 값 있음 (예: 5) - Posts 테이블 컬럼
post.Blog;    // ❌ null - Blogs 테이블 조회 안 함
```

### 왜 FK는 항상 로드될까?

```
┌─────────────────────────────────────────────────────────────┐
│                     Posts 테이블                            │
├─────────┬────────────────────┬────────────────┬────────────┤
│   Id    │       Title        │    Content     │   BlogId   │
├─────────┼────────────────────┼────────────────┼────────────┤
│    1    │  "First Post"      │   "..."        │     5      │ ← BlogId는 이 테이블에 있음
│    2    │  "Second Post"     │   "..."        │     5      │
│    3    │  "Third Post"      │   "..."        │     3      │
└─────────┴────────────────────┴────────────────┴────────────┘

SELECT Id, Title, Content, BlogId FROM Posts
                                   ↑
                          별도 JOIN 없이 조회 가능
```

```
┌─────────────────────────────────────────────────────────────┐
│                     Blogs 테이블                            │
├─────────┬────────────────────┬──────────────────────────────┤
│   Id    │       Name         │           Url                │
├─────────┼────────────────────┼──────────────────────────────┤
│    3    │  "Tech Blog"       │  "https://tech.com"          │
│    5    │  "Dev Blog"        │  "https://dev.com"           │ ← Blog 객체는 여기서 조회
└─────────┴────────────────────┴──────────────────────────────┘

Blog 객체를 가져오려면 JOIN 또는 추가 쿼리 필요
```

---

## N+1 문제 해결 방법

### 방법 1: Eager Loading (Include)

```csharp
// ✅ 해결: Include 사용
var blogs = await context.Blogs
    .Include(b => b.Posts)
    .ToListAsync();

foreach (var blog in blogs)
{
    Console.WriteLine($"Blog: {blog.Name}");
    foreach (var post in blog.Posts)  // 추가 쿼리 없음!
    {
        Console.WriteLine($"  - {post.Title}");
    }
}
```

생성되는 SQL:
```sql
-- 단일 쿼리로 해결!
SELECT [b].[Id], [b].[Name], [b].[Url],
       [p].[Id], [p].[Title], [p].[Content], [p].[BlogId]
FROM [Blogs] AS [b]
LEFT JOIN [Posts] AS [p] ON [b].[Id] = [p].[BlogId]
ORDER BY [b].[Id]
```

### 방법 2: Projection (Select)

```csharp
// ✅ 해결: Projection 사용 (더 효율적)
var blogSummaries = await context.Blogs
    .Select(b => new
    {
        BlogName = b.Name,
        PostTitles = b.Posts.Select(p => p.Title).ToList()
    })
    .ToListAsync();

foreach (var blog in blogSummaries)
{
    Console.WriteLine($"Blog: {blog.BlogName}");
    foreach (var title in blog.PostTitles)
    {
        Console.WriteLine($"  - {title}");
    }
}
```

생성되는 SQL:
```sql
-- 필요한 컬럼만 조회
SELECT [b].[Name], [p].[Title]
FROM [Blogs] AS [b]
LEFT JOIN [Posts] AS [p] ON [b].[Id] = [p].[BlogId]
ORDER BY [b].[Id]
```

### 방법 3: Split Query

```csharp
// ✅ 해결: 분할 쿼리 (여러 컬렉션 Include 시 유용)
var blogs = await context.Blogs
    .Include(b => b.Posts)
    .Include(b => b.Tags)
    .AsSplitQuery()
    .ToListAsync();
```

생성되는 SQL:
```sql
-- 쿼리 1: 블로그
SELECT [b].[Id], [b].[Name], [b].[Url] FROM [Blogs] AS [b]

-- 쿼리 2: 포스트 (IN 절 사용)
SELECT [p].[Id], [p].[Title], [p].[BlogId]
FROM [Posts] AS [p]
WHERE [p].[BlogId] IN (1, 2, 3, 4, 5)

-- 쿼리 3: 태그 (IN 절 사용)
SELECT [t].[Id], [t].[Name], [t].[BlogId]
FROM [Tags] AS [t]
WHERE [t].[BlogId] IN (1, 2, 3, 4, 5)
```

### 방법 4: 명시적 로딩

```csharp
// ✅ 해결: 조건부로 관련 데이터 필요할 때
var blogs = await context.Blogs.ToListAsync();

if (needPosts)
{
    var blogIds = blogs.Select(b => b.Id).ToList();

    await context.Posts
        .Where(p => blogIds.Contains(p.BlogId))
        .LoadAsync();

    // EF Core가 자동으로 Navigation Property 연결
}
```

---

## 해결 방법 비교

### 성능 비교표

| 방법 | 쿼리 수 | 데이터 전송량 | 사용 시나리오 |
|------|--------|-------------|-------------|
| N+1 (문제) | N+1 | 중복 없음 | ❌ 사용 금지 |
| Include | 1 | 중복 있음 | 일반적 사용 |
| Projection | 1 | 최소 | 특정 필드만 필요 |
| Split Query | 2~3 | 중복 없음 | 여러 컬렉션 |

### 선택 가이드

```
필요한 데이터가 무엇인가?
    │
    ├─ 특정 필드만 필요 ──────────────────→ Projection (Select)
    │
    └─ 전체 엔티티 필요
            │
            ├─ 단일 컬렉션 Include ───────→ Include
            │
            └─ 여러 컬렉션 Include
                    │
                    ├─ 데이터가 적음 ────→ Include
                    │
                    └─ 데이터가 많음 ────→ AsSplitQuery
```

---

## N+1 감지 방법

### 방법 1: 로깅 설정

```csharp
// DbContext 설정
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
{
    optionsBuilder
        .UseSqlServer(connectionString)
        .LogTo(Console.WriteLine, LogLevel.Information)
        .EnableSensitiveDataLogging();  // 개발 환경만
}
```

### 방법 2: 쿼리 태그

```csharp
var blogs = await context.Blogs
    .TagWith("GetAllBlogs - Check for N+1")  // SQL에 주석 추가
    .ToListAsync();
```

생성되는 SQL:
```sql
-- GetAllBlogs - Check for N+1
SELECT [b].[Id], [b].[Name], [b].[Url] FROM [Blogs] AS [b]
```

### 방법 3: MiniProfiler

```csharp
// Startup.cs
services.AddMiniProfiler(options =>
{
    options.RouteBasePath = "/profiler";
}).AddEntityFramework();

// 쿼리 수, 실행 시간 등을 웹 UI로 확인 가능
```

### 방법 4: 경고 설정

```csharp
// EF Core 경고를 예외로 변환 (개발 환경)
optionsBuilder.ConfigureWarnings(warnings =>
    warnings.Throw(RelationalEventId.MultipleCollectionIncludeWarning));
```

---

## 실전 예제

### API 엔드포인트에서 N+1 방지

```csharp
// ❌ 문제 코드
[HttpGet]
public async Task<IActionResult> GetOrders()
{
    var orders = await _context.Orders.ToListAsync();

    return Ok(orders.Select(o => new OrderDto
    {
        Id = o.Id,
        CustomerName = o.Customer.Name,  // N+1 발생!
        ItemCount = o.Items.Count         // N+1 발생!
    }));
}

// ✅ 해결 코드 - Include
[HttpGet]
public async Task<IActionResult> GetOrders()
{
    var orders = await _context.Orders
        .Include(o => o.Customer)
        .Include(o => o.Items)
        .ToListAsync();

    return Ok(orders.Select(o => new OrderDto
    {
        Id = o.Id,
        CustomerName = o.Customer.Name,
        ItemCount = o.Items.Count
    }));
}

// ✅ 최적 코드 - Projection
[HttpGet]
public async Task<IActionResult> GetOrders()
{
    var orders = await _context.Orders
        .Select(o => new OrderDto
        {
            Id = o.Id,
            CustomerName = o.Customer.Name,  // 자동 JOIN
            ItemCount = o.Items.Count         // 서브쿼리
        })
        .ToListAsync();

    return Ok(orders);
}
```

### FK만 필요한 경우

```csharp
// ❌ 불필요한 Include
var order = await _context.Orders
    .Include(o => o.Customer)
    .FirstAsync(o => o.Id == orderId);

var customerId = order.Customer.Id;  // Include 했으니 사용

// ✅ FK 직접 사용 (Include 불필요)
var order = await _context.Orders
    .FirstAsync(o => o.Id == orderId);

var customerId = order.CustomerId;  // FK는 항상 로드됨
```

---

## 요약

### N+1 문제 체크리스트

- [ ] `foreach` 안에서 Navigation Property 접근하는지 확인
- [ ] 로그에서 반복되는 SELECT 쿼리 확인
- [ ] Include 또는 Projection 사용 여부 확인

### 핵심 규칙

| 규칙 | 설명 |
|------|------|
| FK는 항상 있다 | `BlogId` 같은 FK Property는 Include 없이 사용 가능 |
| Navigation은 로드 필요 | `Blog` 같은 객체는 Include/Projection 필요 |
| Projection 우선 | 특정 필드만 필요하면 Select 사용 |
| 로깅 필수 | 개발 중 SQL 로그로 N+1 감지 |

## 다음 장 예고

다음 장에서는 Fluent API를 활용한 고급 관계 설정 방법을 알아봅니다.
