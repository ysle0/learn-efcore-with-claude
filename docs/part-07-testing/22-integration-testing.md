# Chapter 22: 통합 테스트

## 개요

실제 데이터베이스를 사용한 통합 테스트 방법을 알아봅니다. TestContainers, 테스트 데이터 설정, 트랜잭션 롤백 전략을 다룹니다.

---

## 21.1 TestContainers 활용

### 설정

```bash
dotnet add package Testcontainers
dotnet add package Testcontainers.MsSql
# 또는
dotnet add package Testcontainers.PostgreSql
```

### SQL Server 컨테이너

```csharp
public class SqlServerFixture : IAsyncLifetime
{
    private readonly MsSqlContainer _container;
    public string ConnectionString => _container.GetConnectionString();

    public SqlServerFixture()
    {
        _container = new MsSqlBuilder()
            .WithImage("mcr.microsoft.com/mssql/server:2022-latest")
            .WithPassword("Strong_password_123!")
            .Build();
    }

    public async Task InitializeAsync()
    {
        await _container.StartAsync();

        // 마이그레이션 적용
        var options = new DbContextOptionsBuilder<ApplicationDbContext>()
            .UseSqlServer(ConnectionString)
            .Options;

        using var context = new ApplicationDbContext(options);
        await context.Database.MigrateAsync();
    }

    public async Task DisposeAsync()
    {
        await _container.DisposeAsync();
    }
}
```

### PostgreSQL 컨테이너

```csharp
public class PostgresFixture : IAsyncLifetime
{
    private readonly PostgreSqlContainer _container;
    public string ConnectionString => _container.GetConnectionString();

    public PostgresFixture()
    {
        _container = new PostgreSqlBuilder()
            .WithImage("postgres:15")
            .WithDatabase("testdb")
            .WithUsername("test")
            .WithPassword("test")
            .Build();
    }

    public async Task InitializeAsync()
    {
        await _container.StartAsync();
    }

    public async Task DisposeAsync()
    {
        await _container.DisposeAsync();
    }
}
```

---

## 21.2 테스트 데이터 설정

### 테스트 데이터 빌더

```csharp
public class TestDataBuilder
{
    private readonly ApplicationDbContext _context;

    public TestDataBuilder(ApplicationDbContext context)
    {
        _context = context;
    }

    public async Task<Category> CreateCategoryAsync(string name = "Test Category")
    {
        var category = new Category { Name = name };
        _context.Categories.Add(category);
        await _context.SaveChangesAsync();
        return category;
    }

    public async Task<Product> CreateProductAsync(
        string name = "Test Product",
        decimal price = 100,
        int? categoryId = null)
    {
        var product = new Product
        {
            Name = name,
            Price = price,
            CategoryId = categoryId ?? (await CreateCategoryAsync()).Id
        };
        _context.Products.Add(product);
        await _context.SaveChangesAsync();
        return product;
    }

    public async Task<Order> CreateOrderWithItemsAsync(int itemCount = 3)
    {
        var order = new Order { OrderDate = DateTime.UtcNow };

        for (int i = 0; i < itemCount; i++)
        {
            var product = await CreateProductAsync($"Product {i}");
            order.OrderItems.Add(new OrderItem
            {
                Product = product,
                Quantity = i + 1,
                UnitPrice = product.Price
            });
        }

        _context.Orders.Add(order);
        await _context.SaveChangesAsync();
        return order;
    }
}
```

### Bogus를 사용한 가짜 데이터

```csharp
// dotnet add package Bogus

public class FakeDataGenerator
{
    private readonly Faker<Product> _productFaker;

    public FakeDataGenerator()
    {
        _productFaker = new Faker<Product>()
            .RuleFor(p => p.Name, f => f.Commerce.ProductName())
            .RuleFor(p => p.Price, f => decimal.Parse(f.Commerce.Price()))
            .RuleFor(p => p.Description, f => f.Commerce.ProductDescription())
            .RuleFor(p => p.CreatedAt, f => f.Date.Past());
    }

    public Product GenerateProduct() => _productFaker.Generate();

    public List<Product> GenerateProducts(int count) =>
        _productFaker.Generate(count);
}
```

---

## 21.3 트랜잭션 롤백 전략

### 테스트 후 롤백

```csharp
public class TransactionalTestBase : IAsyncLifetime
{
    protected ApplicationDbContext Context { get; private set; } = null!;
    private IDbContextTransaction _transaction = null!;

    public async Task InitializeAsync()
    {
        Context = CreateContext();
        _transaction = await Context.Database.BeginTransactionAsync();
    }

    public async Task DisposeAsync()
    {
        await _transaction.RollbackAsync();
        await _transaction.DisposeAsync();
        await Context.DisposeAsync();
    }

    private ApplicationDbContext CreateContext()
    {
        // 실제 DB 연결
        var options = new DbContextOptionsBuilder<ApplicationDbContext>()
            .UseSqlServer("Server=localhost;Database=TestDb;...")
            .Options;

        return new ApplicationDbContext(options);
    }
}

// 사용
public class OrderTests : TransactionalTestBase
{
    [Fact]
    public async Task CreateOrder_ShouldSave()
    {
        // Arrange & Act
        var order = new Order { OrderDate = DateTime.UtcNow };
        Context.Orders.Add(order);
        await Context.SaveChangesAsync();

        // Assert
        Assert.True(order.Id > 0);

        // 테스트 후 자동 롤백됨
    }
}
```

### Respawn으로 데이터베이스 초기화

```csharp
// dotnet add package Respawn

public class DatabaseResetFixture : IAsyncLifetime
{
    private Respawner _respawner = null!;
    public string ConnectionString { get; } = "Server=localhost;Database=TestDb;...";

    public async Task InitializeAsync()
    {
        _respawner = await Respawner.CreateAsync(ConnectionString, new RespawnerOptions
        {
            TablesToIgnore = new[] { "__EFMigrationsHistory" },
            DbAdapter = DbAdapter.SqlServer
        });
    }

    public async Task ResetDatabaseAsync()
    {
        await _respawner.ResetAsync(ConnectionString);
    }

    public Task DisposeAsync() => Task.CompletedTask;
}

// 각 테스트 전에 초기화
public class OrderTests : IAsyncLifetime
{
    private readonly DatabaseResetFixture _fixture;

    public OrderTests(DatabaseResetFixture fixture)
    {
        _fixture = fixture;
    }

    public async Task InitializeAsync()
    {
        await _fixture.ResetDatabaseAsync();
    }

    public Task DisposeAsync() => Task.CompletedTask;
}
```

---

## WebApplicationFactory 통합 테스트

```csharp
public class CustomWebApplicationFactory : WebApplicationFactory<Program>
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureServices(services =>
        {
            // 기존 DbContext 제거
            var descriptor = services.SingleOrDefault(
                d => d.ServiceType == typeof(DbContextOptions<ApplicationDbContext>));

            if (descriptor != null)
                services.Remove(descriptor);

            // 테스트용 DbContext 추가
            services.AddDbContext<ApplicationDbContext>(options =>
                options.UseInMemoryDatabase("TestDb"));
        });
    }
}

public class ApiIntegrationTests : IClassFixture<CustomWebApplicationFactory>
{
    private readonly HttpClient _client;
    private readonly CustomWebApplicationFactory _factory;

    public ApiIntegrationTests(CustomWebApplicationFactory factory)
    {
        _factory = factory;
        _client = factory.CreateClient();
    }

    [Fact]
    public async Task GetProducts_ReturnsSuccess()
    {
        // Arrange
        using var scope = _factory.Services.CreateScope();
        var context = scope.ServiceProvider.GetRequiredService<ApplicationDbContext>();
        context.Products.Add(new Product { Name = "Test", Price = 100 });
        await context.SaveChangesAsync();

        // Act
        var response = await _client.GetAsync("/api/products");

        // Assert
        response.EnsureSuccessStatusCode();
        var products = await response.Content.ReadFromJsonAsync<List<ProductDto>>();
        Assert.NotEmpty(products);
    }
}
```

---

## 요약

| 전략 | 장점 | 단점 | 사용 시나리오 |
|------|------|------|-------------|
| TestContainers | 실제 DB, 격리 | 느린 시작 | CI/CD |
| 트랜잭션 롤백 | 빠름, 격리 | DB 연결 필요 | 로컬 개발 |
| Respawn | 완전 초기화 | 느림 | 상태 초기화 필요 |
| WebApplicationFactory | E2E 테스트 | 복잡한 설정 | API 테스트 |

## 다음 장 예고

다음 Part에서는 Repository 패턴과 실전 아키텍처 패턴을 알아봅니다.
