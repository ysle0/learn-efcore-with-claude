# Chapter 20: 단위 테스트

## 개요

EF Core를 사용하는 코드를 단위 테스트하는 방법을 알아봅니다. InMemory 프로바이더, SQLite In-Memory, 테스트 더블 전략을 다룹니다.

---

## 20.1 InMemory 프로바이더

### 설정

```bash
dotnet add package Microsoft.EntityFrameworkCore.InMemory
```

```csharp
// 테스트용 DbContext 생성
public static ApplicationDbContext CreateInMemoryContext()
{
    var options = new DbContextOptionsBuilder<ApplicationDbContext>()
        .UseInMemoryDatabase(databaseName: Guid.NewGuid().ToString())
        .Options;

    return new ApplicationDbContext(options);
}
```

### 기본 테스트

```csharp
public class ProductServiceTests
{
    [Fact]
    public async Task AddProduct_ShouldSaveToDatabase()
    {
        // Arrange
        using var context = CreateInMemoryContext();
        var service = new ProductService(context);
        var product = new Product { Name = "Test", Price = 100 };

        // Act
        await service.AddProductAsync(product);

        // Assert
        var saved = await context.Products.FirstOrDefaultAsync();
        Assert.NotNull(saved);
        Assert.Equal("Test", saved.Name);
    }

    [Fact]
    public async Task GetProducts_ShouldReturnAllProducts()
    {
        // Arrange
        using var context = CreateInMemoryContext();
        context.Products.AddRange(
            new Product { Name = "P1", Price = 100 },
            new Product { Name = "P2", Price = 200 }
        );
        await context.SaveChangesAsync();

        var service = new ProductService(context);

        // Act
        var products = await service.GetAllProductsAsync();

        // Assert
        Assert.Equal(2, products.Count);
    }
}
```

### InMemory 제한사항

```csharp
// ❌ InMemory에서 지원하지 않는 기능

// 1. 관계형 제약 조건 무시
var orphan = new OrderItem { ProductId = 999 };  // 존재하지 않는 FK
context.OrderItems.Add(orphan);
await context.SaveChangesAsync();  // 성공! (실제 DB에서는 실패)

// 2. 트랜잭션 무시
using var transaction = await context.Database.BeginTransactionAsync();
// InMemory는 트랜잭션을 실제로 처리하지 않음

// 3. Raw SQL 불가
await context.Products.FromSqlRaw("SELECT * FROM Products");  // 예외

// 4. 동시성 토큰 무시
[Timestamp]
public byte[] RowVersion { get; set; }  // 작동하지 않음
```

---

## 20.2 SQLite In-Memory 모드

### 더 정확한 테스트

```csharp
public static ApplicationDbContext CreateSqliteContext()
{
    var connection = new SqliteConnection("DataSource=:memory:");
    connection.Open();  // 연결 유지 필수

    var options = new DbContextOptionsBuilder<ApplicationDbContext>()
        .UseSqlite(connection)
        .Options;

    var context = new ApplicationDbContext(options);
    context.Database.EnsureCreated();  // 스키마 생성

    return context;
}
```

### 테스트 베이스 클래스

```csharp
public abstract class DatabaseTestBase : IDisposable
{
    protected readonly ApplicationDbContext Context;
    private readonly SqliteConnection _connection;

    protected DatabaseTestBase()
    {
        _connection = new SqliteConnection("DataSource=:memory:");
        _connection.Open();

        var options = new DbContextOptionsBuilder<ApplicationDbContext>()
            .UseSqlite(_connection)
            .Options;

        Context = new ApplicationDbContext(options);
        Context.Database.EnsureCreated();
    }

    public void Dispose()
    {
        Context.Dispose();
        _connection.Dispose();
    }
}

// 사용
public class ProductTests : DatabaseTestBase
{
    [Fact]
    public async Task Test_ForeignKey_Constraint()
    {
        // FK 제약 조건이 실제로 적용됨
        var orphan = new OrderItem { ProductId = 999 };
        Context.OrderItems.Add(orphan);

        await Assert.ThrowsAsync<DbUpdateException>(
            () => Context.SaveChangesAsync());
    }
}
```

---

## 20.3 테스트용 DbContext 설정

### Fixture 패턴

```csharp
public class DatabaseFixture : IDisposable
{
    public ApplicationDbContext Context { get; }
    private readonly SqliteConnection _connection;

    public DatabaseFixture()
    {
        _connection = new SqliteConnection("DataSource=:memory:");
        _connection.Open();

        var options = new DbContextOptionsBuilder<ApplicationDbContext>()
            .UseSqlite(_connection)
            .Options;

        Context = new ApplicationDbContext(options);
        Context.Database.EnsureCreated();

        // 공통 테스트 데이터
        SeedTestData();
    }

    private void SeedTestData()
    {
        Context.Categories.AddRange(
            new Category { Id = 1, Name = "Electronics" },
            new Category { Id = 2, Name = "Books" }
        );
        Context.SaveChanges();
    }

    public void Dispose()
    {
        Context.Dispose();
        _connection.Close();
    }
}

// xUnit Collection Fixture
[CollectionDefinition("Database")]
public class DatabaseCollection : ICollectionFixture<DatabaseFixture>
{
}

[Collection("Database")]
public class ProductTests
{
    private readonly DatabaseFixture _fixture;

    public ProductTests(DatabaseFixture fixture)
    {
        _fixture = fixture;
    }

    [Fact]
    public void Test_Categories_Are_Seeded()
    {
        Assert.Equal(2, _fixture.Context.Categories.Count());
    }
}
```

---

## 20.4 Mock vs 실제 데이터베이스

### Repository Mock

```csharp
public interface IProductRepository
{
    Task<Product?> GetByIdAsync(int id);
    Task<List<Product>> GetAllAsync();
    Task AddAsync(Product product);
}

// 테스트에서 Mock 사용
public class OrderServiceTests
{
    [Fact]
    public async Task CreateOrder_ShouldCalculateTotal()
    {
        // Arrange
        var mockRepo = new Mock<IProductRepository>();
        mockRepo.Setup(r => r.GetByIdAsync(1))
            .ReturnsAsync(new Product { Id = 1, Price = 100 });

        var service = new OrderService(mockRepo.Object);

        // Act
        var order = await service.CreateOrderAsync(productId: 1, quantity: 3);

        // Assert
        Assert.Equal(300, order.TotalAmount);
    }
}
```

### 언제 무엇을 사용할까?

```
테스트 전략 선택:

Mock Repository:
✅ 비즈니스 로직 테스트
✅ 빠른 실행
✅ 격리된 테스트
❌ 쿼리 로직 검증 불가

InMemory Provider:
✅ 빠른 실행
✅ 간단한 CRUD 테스트
❌ 관계형 기능 미지원
❌ 쿼리 동작 차이

SQLite In-Memory:
✅ 관계형 기능 지원
✅ 대부분의 쿼리 동작 검증
❌ DB별 기능 차이

실제 데이터베이스:
✅ 100% 정확한 동작
✅ 성능 테스트 가능
❌ 느린 실행
❌ 환경 설정 필요
```

---

## 테스트 헬퍼

```csharp
public static class TestHelper
{
    public static async Task<ApplicationDbContext> CreateSeededContext()
    {
        var context = CreateSqliteContext();

        await context.Products.AddRangeAsync(
            new Product { Name = "Laptop", Price = 1200 },
            new Product { Name = "Phone", Price = 800 }
        );
        await context.SaveChangesAsync();

        // 변경 추적 초기화
        context.ChangeTracker.Clear();

        return context;
    }

    public static ApplicationDbContext CreateSqliteContext()
    {
        var connection = new SqliteConnection("DataSource=:memory:");
        connection.Open();

        var options = new DbContextOptionsBuilder<ApplicationDbContext>()
            .UseSqlite(connection)
            .EnableSensitiveDataLogging()
            .LogTo(Console.WriteLine)
            .Options;

        var context = new ApplicationDbContext(options);
        context.Database.EnsureCreated();

        return context;
    }
}
```

---

## 요약

| 방법 | 속도 | 정확도 | 사용 시나리오 |
|------|------|--------|-------------|
| InMemory | 매우 빠름 | 낮음 | 간단한 CRUD |
| SQLite | 빠름 | 중간 | 관계형 테스트 |
| Mock | 매우 빠름 | - | 비즈니스 로직 |
| 실제 DB | 느림 | 높음 | 통합 테스트 |

## 다음 장 예고

다음 장에서는 TestContainers를 활용한 통합 테스트 방법을 알아봅니다.
