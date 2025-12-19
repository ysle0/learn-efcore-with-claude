# Chapter 11: 쿼리 필터와 전역 필터

## 개요

EF Core의 전역 쿼리 필터를 사용하여 Soft Delete, Multi-Tenancy 등 공통 패턴을 구현하는 방법을 알아봅니다.

---

## 11.1 쿼리 필터(Query Filters)

### 간략 설명
전역 쿼리 필터는 특정 엔티티에 대한 모든 쿼리에 자동으로 적용되는 조건입니다. 모델 레벨에서 정의됩니다.

### 기본 사용법

```csharp
public class Blog
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public bool IsDeleted { get; set; }
}

protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    // 전역 쿼리 필터 설정
    modelBuilder.Entity<Blog>()
        .HasQueryFilter(b => !b.IsDeleted);
}

// 사용 시
var blogs = await context.Blogs.ToListAsync();
// SQL: SELECT * FROM Blogs WHERE IsDeleted = 0

// 필터가 자동으로 모든 쿼리에 적용됨
var blog = await context.Blogs.FirstOrDefaultAsync(b => b.Id == 1);
// SQL: SELECT TOP 1 * FROM Blogs WHERE IsDeleted = 0 AND Id = 1
```

### 필터 무시하기

```csharp
// 특정 쿼리에서 필터 무시
var allBlogs = await context.Blogs
    .IgnoreQueryFilters()
    .ToListAsync();
// SQL: SELECT * FROM Blogs (필터 없음)

// 삭제된 항목만 조회
var deletedBlogs = await context.Blogs
    .IgnoreQueryFilters()
    .Where(b => b.IsDeleted)
    .ToListAsync();
```

### 관련 엔티티에 필터 적용

```csharp
public class Blog
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public bool IsDeleted { get; set; }
    public ICollection<Post> Posts { get; set; } = new List<Post>();
}

public class Post
{
    public int Id { get; set; }
    public string Title { get; set; } = string.Empty;
    public bool IsDeleted { get; set; }
    public int BlogId { get; set; }
    public Blog Blog { get; set; } = null!;
}

protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Blog>().HasQueryFilter(b => !b.IsDeleted);
    modelBuilder.Entity<Post>().HasQueryFilter(p => !p.IsDeleted);
}

// Include 시에도 필터 적용됨
var blogs = await context.Blogs
    .Include(b => b.Posts)
    .ToListAsync();
// 삭제된 Blog와 삭제된 Post 모두 제외됨
```

---

## 11.2 Soft Delete 구현

### 간략 설명
Soft Delete는 데이터를 실제로 삭제하지 않고 삭제된 것으로 표시하는 패턴입니다.

### 기본 구현

```csharp
// 인터페이스 정의
public interface ISoftDelete
{
    bool IsDeleted { get; set; }
    DateTime? DeletedAt { get; set; }
    string? DeletedBy { get; set; }
}

// 엔티티 구현
public class Product : ISoftDelete
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public decimal Price { get; set; }

    // Soft Delete 속성
    public bool IsDeleted { get; set; }
    public DateTime? DeletedAt { get; set; }
    public string? DeletedBy { get; set; }
}
```

### 전역 필터 자동 적용

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    // ISoftDelete를 구현한 모든 엔티티에 필터 적용
    foreach (var entityType in modelBuilder.Model.GetEntityTypes())
    {
        if (typeof(ISoftDelete).IsAssignableFrom(entityType.ClrType))
        {
            var parameter = Expression.Parameter(entityType.ClrType, "e");
            var property = Expression.Property(parameter, nameof(ISoftDelete.IsDeleted));
            var condition = Expression.Equal(property, Expression.Constant(false));
            var lambda = Expression.Lambda(condition, parameter);

            modelBuilder.Entity(entityType.ClrType).HasQueryFilter(lambda);
        }
    }
}
```

### SaveChanges 오버라이드

```csharp
public class ApplicationDbContext : DbContext
{
    private readonly ICurrentUserService _currentUser;

    public ApplicationDbContext(
        DbContextOptions<ApplicationDbContext> options,
        ICurrentUserService currentUser)
        : base(options)
    {
        _currentUser = currentUser;
    }

    public override int SaveChanges()
    {
        HandleSoftDelete();
        return base.SaveChanges();
    }

    public override async Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
    {
        HandleSoftDelete();
        return await base.SaveChangesAsync(cancellationToken);
    }

    private void HandleSoftDelete()
    {
        foreach (var entry in ChangeTracker.Entries<ISoftDelete>())
        {
            if (entry.State == EntityState.Deleted)
            {
                // 실제 삭제 대신 Soft Delete
                entry.State = EntityState.Modified;
                entry.Entity.IsDeleted = true;
                entry.Entity.DeletedAt = DateTime.UtcNow;
                entry.Entity.DeletedBy = _currentUser.UserId;
            }
        }
    }
}

// 사용
context.Products.Remove(product);
await context.SaveChangesAsync();
// DELETE 대신 UPDATE가 실행됨
// UPDATE Products SET IsDeleted = 1, DeletedAt = '...', DeletedBy = '...' WHERE Id = @id
```

### Soft Delete 복원

```csharp
public async Task RestoreProductAsync(int productId)
{
    var product = await context.Products
        .IgnoreQueryFilters()
        .FirstOrDefaultAsync(p => p.Id == productId && p.IsDeleted);

    if (product != null)
    {
        product.IsDeleted = false;
        product.DeletedAt = null;
        product.DeletedBy = null;
        await context.SaveChangesAsync();
    }
}
```

### 영구 삭제

```csharp
public async Task HardDeleteProductAsync(int productId)
{
    var product = await context.Products
        .IgnoreQueryFilters()
        .FirstOrDefaultAsync(p => p.Id == productId);

    if (product != null)
    {
        // 직접 SQL 실행 (SaveChanges 우회)
        await context.Database.ExecuteSqlInterpolatedAsync(
            $"DELETE FROM Products WHERE Id = {productId}");

        // 또는 엔티티 상태를 Detached로 변경 후 삭제
        context.Entry(product).State = EntityState.Detached;

        // Raw SQL로 삭제
        await context.Database.ExecuteSqlInterpolatedAsync(
            $"DELETE FROM Products WHERE Id = {productId}");
    }
}
```

---

## 11.3 Multi-Tenancy 패턴

### 간략 설명
Multi-Tenancy는 여러 테넌트(고객/조직)가 같은 애플리케이션을 공유하면서 데이터를 격리하는 패턴입니다.

### 행 수준 테넌시 (Row-Level)

```csharp
// 테넌트 인터페이스
public interface IHasTenant
{
    string TenantId { get; set; }
}

// 엔티티
public class Product : IHasTenant
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public string TenantId { get; set; } = string.Empty;
}

// 현재 테넌트 서비스
public interface ITenantService
{
    string TenantId { get; }
}

public class TenantService : ITenantService
{
    private readonly IHttpContextAccessor _httpContext;

    public TenantService(IHttpContextAccessor httpContext)
    {
        _httpContext = httpContext;
    }

    public string TenantId =>
        _httpContext.HttpContext?.User?.FindFirst("tenant_id")?.Value
        ?? throw new InvalidOperationException("테넌트 정보가 없습니다.");
}

// DbContext
public class MultiTenantDbContext : DbContext
{
    private readonly ITenantService _tenantService;

    public MultiTenantDbContext(
        DbContextOptions<MultiTenantDbContext> options,
        ITenantService tenantService)
        : base(options)
    {
        _tenantService = tenantService;
    }

    public DbSet<Product> Products => Set<Product>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // 테넌트 필터 적용
        modelBuilder.Entity<Product>()
            .HasQueryFilter(p => p.TenantId == _tenantService.TenantId);

        // 또는 모든 IHasTenant 엔티티에 자동 적용
        foreach (var entityType in modelBuilder.Model.GetEntityTypes())
        {
            if (typeof(IHasTenant).IsAssignableFrom(entityType.ClrType))
            {
                ConfigureTenantFilter(modelBuilder, entityType.ClrType);
            }
        }
    }

    private void ConfigureTenantFilter(ModelBuilder modelBuilder, Type entityType)
    {
        var parameter = Expression.Parameter(entityType, "e");
        var tenantProperty = Expression.Property(parameter, nameof(IHasTenant.TenantId));

        // _tenantService.TenantId 접근
        var tenantService = Expression.Constant(_tenantService);
        var currentTenant = Expression.Property(tenantService, nameof(ITenantService.TenantId));

        var condition = Expression.Equal(tenantProperty, currentTenant);
        var lambda = Expression.Lambda(condition, parameter);

        modelBuilder.Entity(entityType).HasQueryFilter(lambda);
    }

    public override async Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
    {
        // 새 엔티티에 테넌트 ID 자동 설정
        foreach (var entry in ChangeTracker.Entries<IHasTenant>())
        {
            if (entry.State == EntityState.Added)
            {
                entry.Entity.TenantId = _tenantService.TenantId;
            }
        }

        return await base.SaveChangesAsync(cancellationToken);
    }
}
```

### 스키마 기반 테넌시

```csharp
public class SchemaTenantDbContext : DbContext
{
    private readonly ITenantService _tenantService;

    public SchemaTenantDbContext(
        DbContextOptions<SchemaTenantDbContext> options,
        ITenantService tenantService)
        : base(options)
    {
        _tenantService = tenantService;
    }

    public DbSet<Product> Products => Set<Product>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // 테넌트별 스키마 사용
        var schema = _tenantService.TenantId;

        modelBuilder.Entity<Product>().ToTable("Products", schema);
        // 각 테넌트가 자체 스키마를 가짐: tenant1.Products, tenant2.Products
    }
}

// 마이그레이션: 각 테넌트 스키마에 적용 필요
public async Task ApplyMigrationForTenantAsync(string tenantId)
{
    // 테넌트별 스키마 생성
    await context.Database.ExecuteSqlRawAsync(
        $"IF NOT EXISTS (SELECT * FROM sys.schemas WHERE name = '{tenantId}') " +
        $"EXEC('CREATE SCHEMA [{tenantId}]')");

    // 마이그레이션 적용
    await context.Database.MigrateAsync();
}
```

### 데이터베이스 기반 테넌시

```csharp
// 테넌트별 연결 문자열
public class TenantConnectionStringProvider
{
    private readonly IConfiguration _configuration;

    public TenantConnectionStringProvider(IConfiguration configuration)
    {
        _configuration = configuration;
    }

    public string GetConnectionString(string tenantId)
    {
        // 테넌트별 연결 문자열 반환
        return _configuration.GetConnectionString(tenantId)
            ?? throw new InvalidOperationException($"테넌트 '{tenantId}'의 연결 문자열이 없습니다.");
    }
}

// DbContextFactory 사용
public class TenantDbContextFactory : IDbContextFactory<ApplicationDbContext>
{
    private readonly ITenantService _tenantService;
    private readonly TenantConnectionStringProvider _connectionProvider;

    public TenantDbContextFactory(
        ITenantService tenantService,
        TenantConnectionStringProvider connectionProvider)
    {
        _tenantService = tenantService;
        _connectionProvider = connectionProvider;
    }

    public ApplicationDbContext CreateDbContext()
    {
        var connectionString = _connectionProvider.GetConnectionString(_tenantService.TenantId);

        var options = new DbContextOptionsBuilder<ApplicationDbContext>()
            .UseSqlServer(connectionString)
            .Options;

        return new ApplicationDbContext(options);
    }
}
```

### Multi-Tenancy 전략 비교

| 전략 | 격리 수준 | 복잡도 | 비용 | 마이그레이션 |
|------|----------|-------|------|------------|
| 행 수준 | 낮음 | 낮음 | 낮음 | 간단 |
| 스키마 | 중간 | 중간 | 중간 | 중간 |
| 데이터베이스 | 높음 | 높음 | 높음 | 복잡 |

---

## 필터 조합

### 여러 필터 조합

```csharp
public class ApplicationDbContext : DbContext
{
    private readonly ITenantService _tenantService;
    private readonly bool _includeDeleted;

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Soft Delete + Multi-Tenancy 조합
        modelBuilder.Entity<Product>()
            .HasQueryFilter(p =>
                p.TenantId == _tenantService.TenantId &&
                !p.IsDeleted);

        // 동적 필터 (조건부)
        modelBuilder.Entity<Post>()
            .HasQueryFilter(p =>
                (_includeDeleted || !p.IsDeleted) &&
                p.IsPublished);
    }
}

// 주의: 하나의 엔티티에는 하나의 HasQueryFilter만 적용됨
// 여러 조건은 && 연산자로 결합
```

### 필터 비활성화 패턴

```csharp
// 관리자용 컨텍스트 (필터 없음)
public class AdminDbContext : ApplicationDbContext
{
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // 기본 설정 적용 (필터 제외)
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(AdminDbContext).Assembly);

        // 필터를 설정하지 않음
    }
}

// 또는 플래그 사용
public class ApplicationDbContext : DbContext
{
    private readonly bool _applyFilters;

    public ApplicationDbContext(DbContextOptions options, bool applyFilters = true)
        : base(options)
    {
        _applyFilters = applyFilters;
    }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        if (_applyFilters)
        {
            modelBuilder.Entity<Product>()
                .HasQueryFilter(p => !p.IsDeleted);
        }
    }
}
```

---

## 성능 고려사항

### 인덱스 최적화

```csharp
modelBuilder.Entity<Product>(entity =>
{
    // Soft Delete 필터용 인덱스
    entity.HasIndex(p => p.IsDeleted)
        .HasFilter("[IsDeleted] = 0")
        .HasDatabaseName("IX_Products_Active");

    // Multi-Tenancy 필터용 인덱스
    entity.HasIndex(p => new { p.TenantId, p.IsDeleted })
        .HasFilter("[IsDeleted] = 0")
        .HasDatabaseName("IX_Products_Tenant_Active");
});
```

### 쿼리 계획 확인

```csharp
var query = context.Products
    .Where(p => p.Price > 100);

// 생성되는 SQL 확인
var sql = query.ToQueryString();
Console.WriteLine(sql);

// 예상 결과:
// SELECT * FROM Products
// WHERE IsDeleted = 0           -- 전역 필터
//   AND TenantId = @tenantId    -- 테넌트 필터
//   AND Price > 100             -- 사용자 조건
```

---

## 요약

| 패턴 | 사용 시나리오 | 구현 방법 |
|------|-------------|----------|
| 쿼리 필터 | 공통 조건 자동 적용 | HasQueryFilter |
| Soft Delete | 데이터 보존, 감사 | IsDeleted 플래그 |
| Multi-Tenancy | SaaS, 데이터 격리 | TenantId 필터 |
| 필터 무시 | 관리자 기능, 복원 | IgnoreQueryFilters |

## 다음 장 예고

다음 Part에서는 데이터 변경과 저장 방법(Add, Update, Remove, SaveChanges)을 알아봅니다.
