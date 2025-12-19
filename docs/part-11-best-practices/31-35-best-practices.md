# Chapter 31-35: 모범 사례 (Best Practices)

## 개요

EF Core를 효과적으로 사용하기 위한 모범 사례를 알아봅니다. DbContext 관리, 성능, 보안, 구조화 방법을 다룹니다.

---

## 31. DbContext 수명 관리

### 31.1 올바른 수명 주기 선택

```csharp
// ✅ 권장: Scoped (기본값)
services.AddDbContext<ApplicationDbContext>(options =>
    options.UseSqlServer(connectionString));

// 내부적으로 Scoped로 등록됨
// services.AddScoped<ApplicationDbContext>();

// ✅ 권장: DbContext 풀링 (고성능)
services.AddDbContextPool<ApplicationDbContext>(options =>
    options.UseSqlServer(connectionString),
    poolSize: 128);

// ❌ 피해야 함: Singleton
// DbContext는 스레드 안전하지 않음!
// services.AddSingleton<ApplicationDbContext>();  // 위험!

// ❌ 피해야 함: Transient (불필요한 오버헤드)
// services.AddTransient<ApplicationDbContext>();
```

### 31.2 웹 요청과 DbContext

```csharp
// ✅ 권장: 컨트롤러에서 직접 주입
public class ProductsController : ControllerBase
{
    private readonly ApplicationDbContext _context;

    public ProductsController(ApplicationDbContext context)
    {
        _context = context;  // 요청당 하나의 인스턴스
    }

    [HttpGet]
    public async Task<IActionResult> GetProducts()
    {
        var products = await _context.Products.ToListAsync();
        return Ok(products);
    }
}

// ✅ 권장: 서비스 레이어에서 사용
public class ProductService
{
    private readonly ApplicationDbContext _context;

    public ProductService(ApplicationDbContext context)
    {
        _context = context;
    }
}
```

### 31.3 장기 실행 작업에서 DbContext

```csharp
// ❌ 잘못된 예: 장시간 DbContext 유지
public class BackgroundWorker : BackgroundService
{
    private readonly ApplicationDbContext _context;  // 위험!

    public BackgroundWorker(ApplicationDbContext context)
    {
        _context = context;  // Scoped 서비스를 Singleton에서 사용
    }
}

// ✅ 권장: IDbContextFactory 사용
public class BackgroundWorker : BackgroundService
{
    private readonly IDbContextFactory<ApplicationDbContext> _contextFactory;

    public BackgroundWorker(IDbContextFactory<ApplicationDbContext> contextFactory)
    {
        _contextFactory = contextFactory;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            // 각 작업마다 새 컨텍스트
            await using var context = await _contextFactory.CreateDbContextAsync();

            await ProcessBatchAsync(context);
            await Task.Delay(TimeSpan.FromMinutes(1), stoppingToken);
        }
    }

    private async Task ProcessBatchAsync(ApplicationDbContext context)
    {
        var items = await context.Items
            .Where(i => i.NeedsProcessing)
            .Take(100)
            .ToListAsync();

        foreach (var item in items)
        {
            item.IsProcessed = true;
        }

        await context.SaveChangesAsync();
    }
}

// DI 등록
services.AddPooledDbContextFactory<ApplicationDbContext>(options =>
    options.UseSqlServer(connectionString));
```

### 31.4 멀티 테넌트 환경

```csharp
// 테넌트별 연결 문자열
public class TenantDbContext : DbContext
{
    private readonly ITenantService _tenantService;

    public TenantDbContext(
        DbContextOptions<TenantDbContext> options,
        ITenantService tenantService)
        : base(options)
    {
        _tenantService = tenantService;
    }

    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        if (!optionsBuilder.IsConfigured)
        {
            var connectionString = _tenantService.GetConnectionString();
            optionsBuilder.UseSqlServer(connectionString);
        }
    }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // 테넌트 필터 적용
        modelBuilder.Entity<Product>()
            .HasQueryFilter(p => p.TenantId == _tenantService.TenantId);
    }
}
```

---

## 32. 쿼리 성능 최적화

### 32.1 필요한 데이터만 조회

```csharp
// ❌ 모든 컬럼 조회
var products = await context.Products.ToListAsync();

// ✅ 필요한 컬럼만 Projection
var productNames = await context.Products
    .Select(p => new { p.Id, p.Name, p.Price })
    .ToListAsync();

// ✅ DTO로 직접 매핑
var productDtos = await context.Products
    .Select(p => new ProductDto
    {
        Id = p.Id,
        Name = p.Name,
        CategoryName = p.Category.Name
    })
    .ToListAsync();
```

### 32.2 N+1 문제 해결

```csharp
// ❌ N+1 문제 발생
var orders = await context.Orders.ToListAsync();
foreach (var order in orders)
{
    Console.WriteLine(order.Customer.Name);  // 각 주문마다 쿼리!
}

// ✅ Eager Loading
var orders = await context.Orders
    .Include(o => o.Customer)
    .ToListAsync();

// ✅ 또는 Projection
var orderInfo = await context.Orders
    .Select(o => new
    {
        o.Id,
        o.OrderDate,
        CustomerName = o.Customer.Name
    })
    .ToListAsync();
```

### 32.3 AsNoTracking 활용

```csharp
// ✅ 읽기 전용 쿼리에 AsNoTracking
var products = await context.Products
    .AsNoTracking()
    .ToListAsync();

// ✅ 전역 설정
public class ReadOnlyDbContext : ApplicationDbContext
{
    public ReadOnlyDbContext(DbContextOptions options) : base(options)
    {
        ChangeTracker.QueryTrackingBehavior = QueryTrackingBehavior.NoTracking;
        ChangeTracker.AutoDetectChangesEnabled = false;
    }
}

// ✅ 서비스 등록
services.AddDbContext<ReadOnlyDbContext>(options =>
    options.UseSqlServer(connectionString)
           .UseQueryTrackingBehavior(QueryTrackingBehavior.NoTracking));
```

### 32.4 페이지네이션

```csharp
// ✅ 올바른 페이지네이션
public async Task<PagedResult<ProductDto>> GetProductsAsync(int page, int pageSize)
{
    var query = context.Products.AsNoTracking();

    var totalCount = await query.CountAsync();

    var items = await query
        .OrderBy(p => p.Id)  // 반드시 정렬 필요!
        .Skip((page - 1) * pageSize)
        .Take(pageSize)
        .Select(p => new ProductDto { Id = p.Id, Name = p.Name })
        .ToListAsync();

    return new PagedResult<ProductDto>
    {
        Items = items,
        TotalCount = totalCount,
        Page = page,
        PageSize = pageSize
    };
}

// ✅ Keyset Pagination (대용량에 효율적)
public async Task<List<ProductDto>> GetProductsAfterAsync(int lastId, int pageSize)
{
    return await context.Products
        .AsNoTracking()
        .Where(p => p.Id > lastId)
        .OrderBy(p => p.Id)
        .Take(pageSize)
        .Select(p => new ProductDto { Id = p.Id, Name = p.Name })
        .ToListAsync();
}
```

### 32.5 인덱스 활용

```csharp
// ✅ 자주 검색되는 컬럼에 인덱스
modelBuilder.Entity<Product>(entity =>
{
    entity.HasIndex(p => p.CategoryId);  // FK 인덱스
    entity.HasIndex(p => p.Sku).IsUnique();
    entity.HasIndex(p => p.Name);
    entity.HasIndex(p => new { p.CategoryId, p.IsActive });  // 복합 인덱스
});

// ✅ 필터링된 인덱스
modelBuilder.Entity<Product>()
    .HasIndex(p => p.Name)
    .HasFilter("[IsActive] = 1");  // 활성 상품만
```

---

## 33. 트랜잭션 관리

### 33.1 SaveChanges 트랜잭션

```csharp
// SaveChanges는 자동으로 트랜잭션 사용
// 모두 성공하거나 모두 롤백

try
{
    context.Orders.Add(order);
    context.OrderItems.AddRange(items);
    await context.SaveChangesAsync();  // 하나의 트랜잭션
}
catch (Exception)
{
    // 자동 롤백됨
    throw;
}
```

### 33.2 명시적 트랜잭션

```csharp
// ✅ 여러 SaveChanges를 하나의 트랜잭션으로
await using var transaction = await context.Database.BeginTransactionAsync();

try
{
    // 주문 생성
    var order = new Order { CustomerId = customerId };
    context.Orders.Add(order);
    await context.SaveChangesAsync();

    // 재고 감소
    foreach (var item in orderItems)
    {
        var product = await context.Products.FindAsync(item.ProductId);
        product!.StockQuantity -= item.Quantity;
    }
    await context.SaveChangesAsync();

    // 결제 처리
    await paymentService.ProcessAsync(order.Id);

    await transaction.CommitAsync();
}
catch
{
    await transaction.RollbackAsync();
    throw;
}
```

### 33.3 연결 복원력과 트랜잭션

```csharp
// 실행 전략과 트랜잭션 함께 사용
var strategy = context.Database.CreateExecutionStrategy();

await strategy.ExecuteAsync(async () =>
{
    await using var transaction = await context.Database.BeginTransactionAsync();

    try
    {
        // 작업 수행
        await context.SaveChangesAsync();
        await transaction.CommitAsync();
    }
    catch
    {
        await transaction.RollbackAsync();
        throw;
    }
});
```

### 33.4 분산 트랜잭션 대안

```csharp
// ✅ Saga 패턴 사용
public class OrderSaga
{
    public async Task<OrderResult> ExecuteAsync(CreateOrderRequest request)
    {
        var sagaId = Guid.NewGuid();

        try
        {
            // Step 1: 재고 예약
            await _inventoryService.ReserveStockAsync(sagaId, request.Items);

            // Step 2: 결제 처리
            var paymentResult = await _paymentService.ProcessAsync(sagaId, request.Payment);

            if (!paymentResult.Success)
            {
                // 보상 트랜잭션: 재고 예약 취소
                await _inventoryService.ReleaseStockAsync(sagaId);
                return OrderResult.Failed(paymentResult.Error);
            }

            // Step 3: 주문 생성
            var order = await _orderService.CreateAsync(request);

            // Step 4: 재고 확정
            await _inventoryService.ConfirmReservationAsync(sagaId);

            return OrderResult.Success(order);
        }
        catch (Exception ex)
        {
            // 보상 트랜잭션들 실행
            await _inventoryService.ReleaseStockAsync(sagaId);
            await _paymentService.RefundAsync(sagaId);
            throw;
        }
    }
}
```

---

## 34. 보안 모범 사례

### 34.1 SQL Injection 방지

```csharp
// ❌ 위험: 문자열 연결
var query = $"SELECT * FROM Products WHERE Name = '{userInput}'";
var products = await context.Products.FromSqlRaw(query).ToListAsync();

// ✅ 안전: 매개변수화된 쿼리
var products = await context.Products
    .FromSqlInterpolated($"SELECT * FROM Products WHERE Name = {userInput}")
    .ToListAsync();

// ✅ 안전: LINQ 사용 (자동 매개변수화)
var products = await context.Products
    .Where(p => p.Name == userInput)
    .ToListAsync();
```

### 34.2 연결 문자열 보안

```csharp
// ❌ 하드코딩된 연결 문자열
var connectionString = "Server=prod;Database=MyDb;User Id=admin;Password=secret123;";

// ✅ 환경 변수 사용
var connectionString = Environment.GetEnvironmentVariable("DATABASE_CONNECTION");

// ✅ Azure Key Vault 사용
var secretClient = new SecretClient(vaultUri, new DefaultAzureCredential());
var secret = await secretClient.GetSecretAsync("DatabaseConnection");
var connectionString = secret.Value.Value;

// ✅ User Secrets (개발 환경)
// dotnet user-secrets set "ConnectionStrings:Default" "..."

// appsettings.json에서 읽기
var connectionString = configuration.GetConnectionString("Default");
```

### 34.3 민감한 데이터 필터링

```csharp
// ✅ 민감한 데이터 로깅 방지
services.AddDbContext<ApplicationDbContext>(options =>
    options.UseSqlServer(connectionString)
           .EnableSensitiveDataLogging(isDevelopment)  // 개발 환경만
           .LogTo(Console.WriteLine, LogLevel.Information));

// ✅ 출력에서 민감한 필드 제외
public class UserDto
{
    public int Id { get; set; }
    public string Email { get; set; } = string.Empty;

    [JsonIgnore]
    public string PasswordHash { get; set; } = string.Empty;
}

// ✅ 쿼리에서 민감한 필드 제외
var users = await context.Users
    .Select(u => new UserDto
    {
        Id = u.Id,
        Email = u.Email
        // PasswordHash 제외
    })
    .ToListAsync();
```

### 34.4 권한 검사

```csharp
// ✅ 글로벌 필터로 데이터 접근 제어
public class SecureDbContext : DbContext
{
    private readonly ICurrentUserService _currentUser;

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Document>()
            .HasQueryFilter(d =>
                d.OwnerId == _currentUser.UserId ||
                d.SharedWith.Any(s => s.UserId == _currentUser.UserId));
    }
}

// ✅ 서비스 레벨 권한 검사
public async Task<Document?> GetDocumentAsync(int documentId)
{
    var document = await context.Documents.FindAsync(documentId);

    if (document == null) return null;

    if (!await _authorizationService.AuthorizeAsync(_currentUser, document, "Read"))
    {
        throw new UnauthorizedAccessException();
    }

    return document;
}
```

---

## 35. 코드 구조화

### 35.1 DbContext 분리

```csharp
// ✅ 도메인별 DbContext 분리
public class OrderDbContext : DbContext
{
    public DbSet<Order> Orders => Set<Order>();
    public DbSet<OrderItem> OrderItems => Set<OrderItem>();
}

public class InventoryDbContext : DbContext
{
    public DbSet<Product> Products => Set<Product>();
    public DbSet<StockMovement> StockMovements => Set<StockMovement>();
}

public class IdentityDbContext : DbContext
{
    public DbSet<User> Users => Set<User>();
    public DbSet<Role> Roles => Set<Role>();
}
```

### 35.2 설정 파일 분리

```csharp
// ✅ IEntityTypeConfiguration 사용
public class ProductConfiguration : IEntityTypeConfiguration<Product>
{
    public void Configure(EntityTypeBuilder<Product> builder)
    {
        builder.HasKey(p => p.Id);
        builder.Property(p => p.Name).IsRequired().HasMaxLength(200);
        // ...
    }
}

// ✅ 자동 적용
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.ApplyConfigurationsFromAssembly(typeof(ProductConfiguration).Assembly);
}

// ✅ 파일 구조
// Data/
//   Configurations/
//     ProductConfiguration.cs
//     OrderConfiguration.cs
//     CustomerConfiguration.cs
//   ApplicationDbContext.cs
```

### 35.3 Repository 패턴 (선택적)

```csharp
// ✅ 제네릭 Repository
public interface IRepository<T> where T : class
{
    Task<T?> GetByIdAsync(int id);
    Task<IReadOnlyList<T>> GetAllAsync();
    Task AddAsync(T entity);
    void Update(T entity);
    void Delete(T entity);
}

// ✅ 특화된 Repository (복잡한 쿼리)
public interface IOrderRepository : IRepository<Order>
{
    Task<Order?> GetWithItemsAsync(int id);
    Task<IReadOnlyList<Order>> GetByCustomerAsync(int customerId);
    Task<OrderStatistics> GetStatisticsAsync(DateTime from, DateTime to);
}

// ✅ Unit of Work
public interface IUnitOfWork
{
    IProductRepository Products { get; }
    IOrderRepository Orders { get; }
    Task<int> SaveChangesAsync(CancellationToken ct = default);
}
```

### 35.4 CQRS 패턴

```csharp
// ✅ Command와 Query 분리

// Query (읽기 전용)
public class GetProductsQuery : IRequest<List<ProductDto>>
{
    public int? CategoryId { get; set; }
    public string? SearchTerm { get; set; }
}

public class GetProductsQueryHandler : IRequestHandler<GetProductsQuery, List<ProductDto>>
{
    private readonly ReadOnlyDbContext _context;

    public async Task<List<ProductDto>> Handle(
        GetProductsQuery request, CancellationToken ct)
    {
        var query = _context.Products.AsQueryable();

        if (request.CategoryId.HasValue)
            query = query.Where(p => p.CategoryId == request.CategoryId);

        if (!string.IsNullOrEmpty(request.SearchTerm))
            query = query.Where(p => p.Name.Contains(request.SearchTerm));

        return await query
            .Select(p => new ProductDto { Id = p.Id, Name = p.Name })
            .ToListAsync(ct);
    }
}

// Command (쓰기)
public class CreateProductCommand : IRequest<int>
{
    public string Name { get; set; } = string.Empty;
    public decimal Price { get; set; }
}

public class CreateProductCommandHandler : IRequestHandler<CreateProductCommand, int>
{
    private readonly ApplicationDbContext _context;

    public async Task<int> Handle(CreateProductCommand request, CancellationToken ct)
    {
        var product = new Product
        {
            Name = request.Name,
            Price = request.Price
        };

        _context.Products.Add(product);
        await _context.SaveChangesAsync(ct);

        return product.Id;
    }
}
```

---

## 요약

| 영역 | 핵심 권장사항 |
|------|-------------|
| DbContext 수명 | Scoped 또는 Pool 사용, Factory 활용 |
| 쿼리 성능 | Projection, AsNoTracking, 인덱스 |
| 트랜잭션 | 명시적 트랜잭션, 복원력 전략 |
| 보안 | 매개변수화 쿼리, 연결 문자열 보호 |
| 코드 구조 | 설정 분리, 적절한 패턴 적용 |

## 다음 Part 예고

다음 Part에서는 EF Core 사용 시 흔히 발생하는 실수들과 해결 방법을 알아봅니다.
