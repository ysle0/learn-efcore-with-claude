# Chapter 22: Repository 패턴

## 개요

Repository 패턴을 사용하여 데이터 접근 로직을 추상화하는 방법을 알아봅니다. Generic Repository, Unit of Work, 그리고 패턴의 장단점을 다룹니다.

---

## 22.1 Generic Repository

### 기본 구현

```csharp
public interface IRepository<T> where T : class
{
    Task<T?> GetByIdAsync(int id);
    Task<IEnumerable<T>> GetAllAsync();
    Task<IEnumerable<T>> FindAsync(Expression<Func<T, bool>> predicate);
    Task AddAsync(T entity);
    Task AddRangeAsync(IEnumerable<T> entities);
    void Update(T entity);
    void Remove(T entity);
    void RemoveRange(IEnumerable<T> entities);
}

public class Repository<T> : IRepository<T> where T : class
{
    protected readonly DbContext _context;
    protected readonly DbSet<T> _dbSet;

    public Repository(DbContext context)
    {
        _context = context;
        _dbSet = context.Set<T>();
    }

    public async Task<T?> GetByIdAsync(int id)
        => await _dbSet.FindAsync(id);

    public async Task<IEnumerable<T>> GetAllAsync()
        => await _dbSet.ToListAsync();

    public async Task<IEnumerable<T>> FindAsync(Expression<Func<T, bool>> predicate)
        => await _dbSet.Where(predicate).ToListAsync();

    public async Task AddAsync(T entity)
        => await _dbSet.AddAsync(entity);

    public async Task AddRangeAsync(IEnumerable<T> entities)
        => await _dbSet.AddRangeAsync(entities);

    public void Update(T entity)
        => _dbSet.Update(entity);

    public void Remove(T entity)
        => _dbSet.Remove(entity);

    public void RemoveRange(IEnumerable<T> entities)
        => _dbSet.RemoveRange(entities);
}
```

### 특정 Repository

```csharp
public interface IProductRepository : IRepository<Product>
{
    Task<IEnumerable<Product>> GetByCategoryAsync(int categoryId);
    Task<IEnumerable<Product>> GetActiveProductsAsync();
    Task<Product?> GetWithDetailsAsync(int id);
}

public class ProductRepository : Repository<Product>, IProductRepository
{
    public ProductRepository(ApplicationDbContext context) : base(context)
    {
    }

    public async Task<IEnumerable<Product>> GetByCategoryAsync(int categoryId)
        => await _dbSet
            .Where(p => p.CategoryId == categoryId)
            .ToListAsync();

    public async Task<IEnumerable<Product>> GetActiveProductsAsync()
        => await _dbSet
            .Where(p => p.IsActive)
            .OrderBy(p => p.Name)
            .ToListAsync();

    public async Task<Product?> GetWithDetailsAsync(int id)
        => await _dbSet
            .Include(p => p.Category)
            .Include(p => p.Reviews)
            .FirstOrDefaultAsync(p => p.Id == id);
}
```

---

## 22.2 Unit of Work 패턴

```csharp
public interface IUnitOfWork : IDisposable
{
    IProductRepository Products { get; }
    ICategoryRepository Categories { get; }
    IOrderRepository Orders { get; }

    Task<int> SaveChangesAsync();
    Task BeginTransactionAsync();
    Task CommitAsync();
    Task RollbackAsync();
}

public class UnitOfWork : IUnitOfWork
{
    private readonly ApplicationDbContext _context;
    private IDbContextTransaction? _transaction;

    public IProductRepository Products { get; }
    public ICategoryRepository Categories { get; }
    public IOrderRepository Orders { get; }

    public UnitOfWork(ApplicationDbContext context)
    {
        _context = context;
        Products = new ProductRepository(context);
        Categories = new CategoryRepository(context);
        Orders = new OrderRepository(context);
    }

    public async Task<int> SaveChangesAsync()
        => await _context.SaveChangesAsync();

    public async Task BeginTransactionAsync()
        => _transaction = await _context.Database.BeginTransactionAsync();

    public async Task CommitAsync()
    {
        if (_transaction != null)
            await _transaction.CommitAsync();
    }

    public async Task RollbackAsync()
    {
        if (_transaction != null)
            await _transaction.RollbackAsync();
    }

    public void Dispose()
    {
        _transaction?.Dispose();
        _context.Dispose();
    }
}
```

### 사용 예시

```csharp
public class OrderService
{
    private readonly IUnitOfWork _unitOfWork;

    public OrderService(IUnitOfWork unitOfWork)
    {
        _unitOfWork = unitOfWork;
    }

    public async Task CreateOrderAsync(CreateOrderDto dto)
    {
        await _unitOfWork.BeginTransactionAsync();

        try
        {
            var product = await _unitOfWork.Products.GetByIdAsync(dto.ProductId);
            if (product == null)
                throw new NotFoundException("Product not found");

            var order = new Order
            {
                ProductId = dto.ProductId,
                Quantity = dto.Quantity,
                TotalAmount = product.Price * dto.Quantity
            };

            await _unitOfWork.Orders.AddAsync(order);
            await _unitOfWork.SaveChangesAsync();
            await _unitOfWork.CommitAsync();
        }
        catch
        {
            await _unitOfWork.RollbackAsync();
            throw;
        }
    }
}
```

---

## 22.3 Repository 패턴의 장단점

### 장점

```csharp
// 1. 테스트 용이성
public class OrderServiceTests
{
    [Fact]
    public async Task CreateOrder_ShouldWork()
    {
        var mockUoW = new Mock<IUnitOfWork>();
        mockUoW.Setup(u => u.Products.GetByIdAsync(1))
            .ReturnsAsync(new Product { Id = 1, Price = 100 });

        var service = new OrderService(mockUoW.Object);
        // 실제 DB 없이 테스트 가능
    }
}

// 2. 데이터 접근 로직 중앙화
public async Task<IEnumerable<Product>> GetActiveProductsAsync()
    => await _dbSet
        .Where(p => p.IsActive && !p.IsDeleted)
        .Include(p => p.Category)
        .OrderBy(p => p.Name)
        .ToListAsync();
// 같은 쿼리 로직을 여러 곳에서 재사용
```

### 단점

```csharp
// 1. 추가 추상화 레이어
// DbContext 자체가 이미 Repository + UoW 패턴 구현

// 2. EF Core 기능 제한
// Include, AsNoTracking 등 직접 노출 어려움

// 3. 누수되는 추상화
public interface IProductRepository
{
    // EF Core 특정 타입 노출
    IQueryable<Product> GetQueryable();  // 추상화 누수
}
```

---

## 22.4 직접 DbContext 사용 vs Repository

### DbContext 직접 사용 (권장하는 경우)

```csharp
public class ProductService
{
    private readonly ApplicationDbContext _context;

    public ProductService(ApplicationDbContext context)
    {
        _context = context;
    }

    public async Task<List<ProductDto>> GetProductsAsync(ProductFilter filter)
    {
        var query = _context.Products.AsQueryable();

        if (filter.CategoryId.HasValue)
            query = query.Where(p => p.CategoryId == filter.CategoryId);

        if (!string.IsNullOrEmpty(filter.SearchTerm))
            query = query.Where(p => p.Name.Contains(filter.SearchTerm));

        return await query
            .Select(p => new ProductDto { ... })
            .ToListAsync();
    }
}

// 장점:
// - 간단함
// - EF Core 전체 기능 활용
// - 직관적인 코드
```

### Repository 사용 (권장하는 경우)

```csharp
// 1. 복잡한 도메인 로직이 있을 때
// 2. 여러 데이터 소스를 다룰 때
// 3. 캐싱 등 추가 로직이 필요할 때

public class CachedProductRepository : IProductRepository
{
    private readonly IProductRepository _inner;
    private readonly IMemoryCache _cache;

    public async Task<Product?> GetByIdAsync(int id)
    {
        var key = $"product_{id}";
        return await _cache.GetOrCreateAsync(key, async entry =>
        {
            entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10);
            return await _inner.GetByIdAsync(id);
        });
    }
}
```

---

## 요약

| 패턴 | 사용 시나리오 | 복잡도 |
|------|-------------|--------|
| DbContext 직접 | 간단한 앱, CRUD | 낮음 |
| Generic Repository | 공통 CRUD 추상화 | 중간 |
| Specific Repository | 복잡한 쿼리 로직 | 중간 |
| Unit of Work | 트랜잭션 관리 | 높음 |

## 다음 장 예고

다음 장에서는 DDD(Domain-Driven Design)와 EF Core의 통합을 알아봅니다.
