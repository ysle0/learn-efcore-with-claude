# Chapter 09: 관련 데이터 로딩

## 개요

네비게이션 속성을 통해 관련 데이터를 로딩하는 세 가지 전략을 알아봅니다. 각 전략의 특징과 적합한 사용 시나리오를 상세히 다룹니다.

---

## 로딩 전략 개요

```
┌─────────────────────────────────────────────────────────────────┐
│                    관련 데이터 로딩 전략                          │
├─────────────────┬─────────────────┬─────────────────────────────┤
│   즉시 로딩      │    지연 로딩     │        명시적 로딩           │
│ (Eager Loading) │ (Lazy Loading)  │   (Explicit Loading)        │
├─────────────────┼─────────────────┼─────────────────────────────┤
│ Include 사용     │ 자동 로딩       │ Load() 수동 호출            │
│ 쿼리 시 함께 로드 │ 접근 시 로드    │ 필요할 때 로드               │
│ N+1 문제 방지    │ N+1 문제 발생   │ 선택적 로딩                  │
└─────────────────┴─────────────────┴─────────────────────────────┘
```

---

## 9.1 즉시 로딩(Eager Loading) - Include/ThenInclude

### 간략 설명
쿼리 실행 시 관련 데이터를 함께 로드합니다. Include 메서드를 사용하며, N+1 문제를 방지하는 권장 방식입니다.

### 기본 사용법

```csharp
// 엔티티 구조
public class Blog
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public ICollection<Post> Posts { get; set; } = new List<Post>();
}

public class Post
{
    public int Id { get; set; }
    public string Title { get; set; } = string.Empty;
    public int BlogId { get; set; }
    public Blog Blog { get; set; } = null!;
    public ICollection<Comment> Comments { get; set; } = new List<Comment>();
    public Author Author { get; set; } = null!;
}

// 단일 Include
var blogs = await context.Blogs
    .Include(b => b.Posts)
    .ToListAsync();

// 생성되는 SQL
// SELECT b.*, p.*
// FROM Blogs b
// LEFT JOIN Posts p ON b.Id = p.BlogId
```

### ThenInclude - 중첩 관계

```csharp
// 다단계 Include
var blogs = await context.Blogs
    .Include(b => b.Posts)
        .ThenInclude(p => p.Comments)
    .ToListAsync();

// Blog → Posts → Comments 모두 로드

// 여러 경로 Include
var blogs = await context.Blogs
    .Include(b => b.Posts)
        .ThenInclude(p => p.Comments)
    .Include(b => b.Posts)
        .ThenInclude(p => p.Author)
    .ToListAsync();

// 같은 레벨의 여러 관계
var posts = await context.Posts
    .Include(p => p.Blog)
    .Include(p => p.Author)
    .Include(p => p.Comments)
    .Include(p => p.Tags)
    .ToListAsync();
```

### 필터링된 Include (EF Core 5+)

```csharp
// 관련 데이터 필터링
var blogs = await context.Blogs
    .Include(b => b.Posts.Where(p => p.IsPublished))
    .ToListAsync();

// 정렬
var blogs = await context.Blogs
    .Include(b => b.Posts.OrderByDescending(p => p.PublishedAt))
    .ToListAsync();

// 개수 제한
var blogs = await context.Blogs
    .Include(b => b.Posts
        .Where(p => p.IsPublished)
        .OrderByDescending(p => p.PublishedAt)
        .Take(5))
    .ToListAsync();

// 중첩에서도 가능
var blogs = await context.Blogs
    .Include(b => b.Posts.Where(p => p.IsPublished))
        .ThenInclude(p => p.Comments.Where(c => !c.IsSpam))
    .ToListAsync();
```

### 문자열 기반 Include

```csharp
// 동적 Include (런타임에 결정)
var blogs = await context.Blogs
    .Include("Posts")
    .Include("Posts.Comments")
    .ToListAsync();

// 사용 시나리오: 조건부 Include
string includeProperty = userWantsPosts ? "Posts" : null;
var query = context.Blogs.AsQueryable();

if (!string.IsNullOrEmpty(includeProperty))
{
    query = query.Include(includeProperty);
}

var blogs = await query.ToListAsync();
```

### AutoInclude (EF Core 6+)

```csharp
// 자동 Include 설정
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Post>()
        .Navigation(p => p.Author)
        .AutoInclude();  // 항상 Author 포함
}

// 이제 Post 조회 시 항상 Author 포함
var posts = await context.Posts.ToListAsync();
// Author가 자동으로 로드됨

// AutoInclude 무시
var posts = await context.Posts
    .IgnoreAutoIncludes()
    .ToListAsync();
```

---

## 9.2 지연 로딩(Lazy Loading)

### 간략 설명
네비게이션 속성에 처음 접근할 때 자동으로 데이터를 로드합니다. 편리하지만 N+1 문제를 일으킬 수 있습니다.

### 설정 방법

```bash
# 패키지 설치
dotnet add package Microsoft.EntityFrameworkCore.Proxies
```

```csharp
// DbContext 설정
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
{
    optionsBuilder
        .UseLazyLoadingProxies()  // 프록시 기반 지연 로딩
        .UseSqlServer(connectionString);
}

// 또는 DI 설정
services.AddDbContext<BloggingContext>(options =>
    options
        .UseLazyLoadingProxies()
        .UseSqlServer(connectionString));
```

### 엔티티 요구사항

```csharp
// 지연 로딩을 위한 조건:
// 1. 네비게이션 속성이 virtual
// 2. 클래스가 sealed가 아님

public class Blog
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;

    // virtual 키워드 필수
    public virtual ICollection<Post> Posts { get; set; } = new List<Post>();
}

public class Post
{
    public int Id { get; set; }
    public string Title { get; set; } = string.Empty;

    public int BlogId { get; set; }
    public virtual Blog Blog { get; set; } = null!;  // virtual 필수
}
```

### 사용 예제

```csharp
// 지연 로딩 동작
var blog = await context.Blogs.FirstAsync();
// SQL: SELECT TOP 1 * FROM Blogs

foreach (var post in blog.Posts)  // 여기서 추가 쿼리 발생!
{
    // SQL: SELECT * FROM Posts WHERE BlogId = @blogId
    Console.WriteLine(post.Title);
}
```

### N+1 문제

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

// 블로그가 100개면 → 101번의 쿼리 실행!

// ✅ 해결책: 즉시 로딩 사용
var blogs = await context.Blogs
    .Include(b => b.Posts)
    .ToListAsync();  // 1번 쿼리 (JOIN)
```

### 프록시 없는 지연 로딩

```csharp
// ILazyLoader 주입 방식
public class Blog
{
    private readonly ILazyLoader _lazyLoader;
    private ICollection<Post> _posts;

    public Blog() { }

    public Blog(ILazyLoader lazyLoader)
    {
        _lazyLoader = lazyLoader;
    }

    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;

    public ICollection<Post> Posts
    {
        get => _lazyLoader.Load(this, ref _posts);
        set => _posts = value;
    }
}

// Action<object, string> 방식 (더 가벼움)
public class Blog
{
    private readonly Action<object, string> _lazyLoader;
    private ICollection<Post> _posts;

    public Blog() { }

    public Blog(Action<object, string> lazyLoader)
    {
        _lazyLoader = lazyLoader;
    }

    public ICollection<Post> Posts
    {
        get
        {
            _lazyLoader?.Invoke(this, nameof(Posts));
            return _posts;
        }
        set => _posts = value;
    }
}
```

---

## 9.3 명시적 로딩(Explicit Loading)

### 간략 설명
필요한 시점에 명시적으로 관련 데이터를 로드합니다. 조건부 로딩이나 지연 로딩을 사용할 수 없는 경우에 유용합니다.

### 기본 사용법

```csharp
// 엔티티 로드 후 관련 데이터 로드
var blog = await context.Blogs.FirstAsync();

// 컬렉션 로드
await context.Entry(blog)
    .Collection(b => b.Posts)
    .LoadAsync();

// 참조 로드
var post = await context.Posts.FirstAsync();
await context.Entry(post)
    .Reference(p => p.Blog)
    .LoadAsync();
```

### 필터링된 명시적 로딩

```csharp
// 조건부 로딩
var blog = await context.Blogs.FirstAsync();

await context.Entry(blog)
    .Collection(b => b.Posts)
    .Query()
    .Where(p => p.IsPublished)
    .LoadAsync();

// 정렬된 로딩
await context.Entry(blog)
    .Collection(b => b.Posts)
    .Query()
    .OrderByDescending(p => p.PublishedAt)
    .Take(10)
    .LoadAsync();

// 중첩 Include와 함께
await context.Entry(blog)
    .Collection(b => b.Posts)
    .Query()
    .Include(p => p.Comments)
    .Where(p => p.IsPublished)
    .LoadAsync();
```

### 로드 여부 확인

```csharp
// 이미 로드되었는지 확인
var blog = await context.Blogs.FirstAsync();

bool isLoaded = context.Entry(blog)
    .Collection(b => b.Posts)
    .IsLoaded;

if (!isLoaded)
{
    await context.Entry(blog)
        .Collection(b => b.Posts)
        .LoadAsync();
}

// 조건부 로딩 패턴
public async Task<Blog> GetBlogWithPostsIfNeededAsync(int blogId, bool includePosts)
{
    var blog = await context.Blogs.FindAsync(blogId);

    if (blog != null && includePosts)
    {
        await context.Entry(blog)
            .Collection(b => b.Posts)
            .LoadAsync();
    }

    return blog;
}
```

### 명시적 로딩 활용 사례

```csharp
// 사용 사례 1: 조건부 로딩
public async Task<Order> GetOrderAsync(int orderId, bool includeItems)
{
    var order = await context.Orders.FindAsync(orderId);

    if (order != null && includeItems)
    {
        await context.Entry(order)
            .Collection(o => o.OrderItems)
            .Query()
            .Include(oi => oi.Product)
            .LoadAsync();
    }

    return order;
}

// 사용 사례 2: 페이징된 관련 데이터
public async Task<(Blog Blog, List<Post> Posts, int TotalPosts)>
    GetBlogWithPagedPostsAsync(int blogId, int page, int pageSize)
{
    var blog = await context.Blogs.FindAsync(blogId);

    var totalPosts = await context.Entry(blog)
        .Collection(b => b.Posts)
        .Query()
        .CountAsync();

    var posts = await context.Entry(blog)
        .Collection(b => b.Posts)
        .Query()
        .OrderByDescending(p => p.PublishedAt)
        .Skip((page - 1) * pageSize)
        .Take(pageSize)
        .ToListAsync();

    return (blog, posts, totalPosts);
}
```

---

## 9.4 분할 쿼리(Split Queries)

### 간략 설명
EF Core 5+에서 도입된 기능으로, 단일 쿼리 대신 여러 쿼리로 데이터를 로드합니다. 카테시안 곱 문제를 해결합니다.

### 카테시안 곱 문제

```csharp
// 문제 상황: 여러 컬렉션 Include
var blogs = await context.Blogs
    .Include(b => b.Posts)
    .Include(b => b.Tags)
    .ToListAsync();

// 생성되는 SQL (단일 쿼리)
// SELECT b.*, p.*, t.*
// FROM Blogs b
// LEFT JOIN Posts p ON b.Id = p.BlogId
// LEFT JOIN Tags t ON b.Id = t.BlogId

// 문제: 결과 행 수 = Blogs × Posts × Tags (카테시안 곱)
// 블로그 1개, 포스트 10개, 태그 5개 → 50행 반환!
```

### 분할 쿼리 사용

```csharp
// 분할 쿼리 (권장)
var blogs = await context.Blogs
    .Include(b => b.Posts)
    .Include(b => b.Tags)
    .AsSplitQuery()
    .ToListAsync();

// 생성되는 SQL (3개의 쿼리)
// 쿼리 1: SELECT * FROM Blogs
// 쿼리 2: SELECT * FROM Posts WHERE BlogId IN (1, 2, 3...)
// 쿼리 3: SELECT * FROM Tags WHERE BlogId IN (1, 2, 3...)

// 장점: 중복 데이터 없음, 데이터 전송량 감소
```

### 전역 설정

```csharp
// DbContext 수준에서 기본값 설정
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
{
    optionsBuilder.UseSqlServer(
        connectionString,
        o => o.UseQuerySplittingBehavior(QuerySplittingBehavior.SplitQuery));
}

// 개별 쿼리에서 단일 쿼리 강제
var blogs = await context.Blogs
    .Include(b => b.Posts)
    .AsSingleQuery()
    .ToListAsync();
```

### 분할 쿼리 주의사항

```csharp
// ⚠️ 주의: 정합성 문제
// 여러 쿼리 사이에 데이터가 변경될 수 있음

// 트랜잭션으로 해결
using var transaction = await context.Database.BeginTransactionAsync();

var blogs = await context.Blogs
    .Include(b => b.Posts)
    .AsSplitQuery()
    .ToListAsync();

await transaction.CommitAsync();

// 정렬 시 주의
// 분할 쿼리는 각 쿼리에서 독립적으로 정렬됨
// 전체 결과의 정렬 순서가 달라질 수 있음
```

### 언제 분할 쿼리를 사용할까?

```
분할 쿼리 선택 기준:

사용 권장:
✅ 여러 컬렉션 Include 시
✅ 관련 데이터가 많을 때
✅ 카테시안 곱으로 인한 데이터 중복이 심할 때

사용 비권장:
❌ 단일 컬렉션 Include
❌ 실시간 정합성이 중요한 경우
❌ 정렬 순서가 중요한 경우
❌ 데이터베이스 왕복을 최소화해야 할 때
```

---

## 로딩 전략 비교 및 선택

### 비교 표

| 전략 | 쿼리 수 | N+1 문제 | 사용 편의성 | 제어력 |
|------|--------|---------|-----------|-------|
| 즉시 로딩 | 1 (또는 Split) | 없음 | 명시적 | 높음 |
| 지연 로딩 | N+1 | 있음 | 자동 | 낮음 |
| 명시적 로딩 | 필요한 만큼 | 제어 가능 | 수동 | 매우 높음 |

### 권장 사용 패턴

```csharp
// ✅ 권장: 즉시 로딩 (알려진 요구사항)
public async Task<List<OrderDto>> GetOrdersWithDetailsAsync()
{
    return await context.Orders
        .Include(o => o.Customer)
        .Include(o => o.OrderItems)
            .ThenInclude(oi => oi.Product)
        .Select(o => new OrderDto
        {
            OrderId = o.Id,
            CustomerName = o.Customer.Name,
            Items = o.OrderItems.Select(oi => new OrderItemDto
            {
                ProductName = oi.Product.Name,
                Quantity = oi.Quantity
            }).ToList()
        })
        .ToListAsync();
}

// ✅ 권장: 명시적 로딩 (조건부 요구사항)
public async Task<Order> GetOrderAsync(int id, bool includeDetails)
{
    var order = await context.Orders.FindAsync(id);

    if (order != null && includeDetails)
    {
        await context.Entry(order)
            .Collection(o => o.OrderItems)
            .Query()
            .Include(oi => oi.Product)
            .LoadAsync();
    }

    return order;
}

// ⚠️ 주의: 지연 로딩 (예측하기 어려움)
// 사용하려면 성능 프로파일링 필수
```

---

## 요약

| 전략 | 사용 시나리오 | 주의사항 |
|------|-------------|---------|
| 즉시 로딩 | 관련 데이터가 항상 필요 | 과도한 Include 주의 |
| 지연 로딩 | 관련 데이터 필요 여부 불확실 | N+1 문제 |
| 명시적 로딩 | 조건부 로딩 필요 | 추가 코드 필요 |
| 분할 쿼리 | 여러 컬렉션 Include | 정합성 주의 |

## 다음 장 예고

다음 장에서는 GroupBy, 조인, 서브쿼리 등 고급 쿼리 기법을 알아봅니다.
