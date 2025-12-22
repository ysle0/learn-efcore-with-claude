# Chapter 24-25: DDD와 Clean Architecture

## 개요

Domain-Driven Design과 Clean Architecture 패턴을 EF Core와 함께 사용하는 방법을 알아봅니다.

---

## 24. DDD와 EF Core

### Aggregate Root

```csharp
// 집합 루트 기본 클래스
public abstract class AggregateRoot
{
    private readonly List<IDomainEvent> _domainEvents = new();
    public IReadOnlyCollection<IDomainEvent> DomainEvents => _domainEvents.AsReadOnly();

    protected void AddDomainEvent(IDomainEvent domainEvent)
        => _domainEvents.Add(domainEvent);

    public void ClearDomainEvents() => _domainEvents.Clear();
}

// Order Aggregate
public class Order : AggregateRoot
{
    public int Id { get; private set; }
    public OrderStatus Status { get; private set; }

    private readonly List<OrderItem> _items = new();
    public IReadOnlyCollection<OrderItem> Items => _items.AsReadOnly();

    private Order() { }  // EF Core용

    public static Order Create(int customerId)
    {
        var order = new Order
        {
            Status = OrderStatus.Pending
        };
        order.AddDomainEvent(new OrderCreatedEvent(order.Id));
        return order;
    }

    public void AddItem(int productId, int quantity, decimal price)
    {
        var item = new OrderItem(productId, quantity, price);
        _items.Add(item);
    }

    public void Confirm()
    {
        if (Status != OrderStatus.Pending)
            throw new InvalidOperationException("이미 처리된 주문입니다.");

        Status = OrderStatus.Confirmed;
        AddDomainEvent(new OrderConfirmedEvent(Id));
    }
}
```

### 값 객체 매핑

```csharp
public class Money
{
    public decimal Amount { get; }
    public string Currency { get; }

    public Money(decimal amount, string currency)
    {
        Amount = amount;
        Currency = currency;
    }
}

// Fluent API
modelBuilder.Entity<Product>()
    .OwnsOne(p => p.Price, price =>
    {
        price.Property(m => m.Amount).HasColumnName("Price");
        price.Property(m => m.Currency).HasColumnName("PriceCurrency");
    });
```

### 도메인 이벤트 발행

```csharp
public class DomainEventDispatcher
{
    private readonly IServiceProvider _serviceProvider;

    public async Task DispatchEventsAsync(DbContext context)
    {
        var aggregates = context.ChangeTracker.Entries<AggregateRoot>()
            .Select(e => e.Entity)
            .Where(e => e.DomainEvents.Any())
            .ToList();

        var events = aggregates.SelectMany(a => a.DomainEvents).ToList();

        aggregates.ForEach(a => a.ClearDomainEvents());

        foreach (var domainEvent in events)
        {
            var handlerType = typeof(IDomainEventHandler<>)
                .MakeGenericType(domainEvent.GetType());
            var handlers = _serviceProvider.GetServices(handlerType);

            foreach (dynamic handler in handlers)
            {
                await handler.HandleAsync((dynamic)domainEvent);
            }
        }
    }
}
```

---

## 25. Clean Architecture와 EF Core

### 프로젝트 구조

```
Solution/
├── Domain/                    # 엔티티, 값 객체, 인터페이스
│   ├── Entities/
│   ├── ValueObjects/
│   └── Interfaces/
├── Application/               # 유스케이스, DTO
│   ├── Common/
│   ├── Products/
│   └── Orders/
├── Infrastructure/            # EF Core, 외부 서비스
│   ├── Persistence/
│   │   ├── DbContext.cs
│   │   └── Configurations/
│   └── Services/
└── WebApi/                    # 컨트롤러, 미들웨어
```

### 계층별 구현

```csharp
// Domain Layer - 인터페이스 정의
public interface IProductRepository
{
    Task<Product?> GetByIdAsync(int id, CancellationToken cancellationToken);
    Task AddAsync(Product product, CancellationToken cancellationToken);
}

// Application Layer - 유스케이스
public class GetProductQuery : IRequest<ProductDto>
{
    public int ProductId { get; set; }
}

public class GetProductQueryHandler : IRequestHandler<GetProductQuery, ProductDto>
{
    private readonly IProductRepository _repository;

    public async Task<ProductDto> Handle(
        GetProductQuery request,
        CancellationToken cancellationToken)
    {
        var product = await _repository.GetByIdAsync(
            request.ProductId, cancellationToken);

        return product == null ? null : new ProductDto(product);
    }
}

// Infrastructure Layer - 구현
public class ProductRepository : IProductRepository
{
    private readonly ApplicationDbContext _context;

    public async Task<Product?> GetByIdAsync(
        int id, CancellationToken cancellationToken)
    {
        return await _context.Products
            .Include(p => p.Category)
            .FirstOrDefaultAsync(p => p.Id == id, cancellationToken);
    }

    public async Task AddAsync(Product product, CancellationToken cancellationToken)
    {
        await _context.Products.AddAsync(product, cancellationToken);
    }
}
```

### Specification 패턴

```csharp
public abstract class Specification<T>
{
    public abstract Expression<Func<T, bool>> ToExpression();

    public bool IsSatisfiedBy(T entity)
    {
        var predicate = ToExpression().Compile();
        return predicate(entity);
    }
}

public class ActiveProductSpec : Specification<Product>
{
    public override Expression<Func<Product, bool>> ToExpression()
        => p => p.IsActive && !p.IsDeleted;
}

public class ProductByCategorySpec : Specification<Product>
{
    private readonly int _categoryId;

    public ProductByCategorySpec(int categoryId)
    {
        _categoryId = categoryId;
    }

    public override Expression<Func<Product, bool>> ToExpression()
        => p => p.CategoryId == _categoryId;
}

// 사용
var spec = new ActiveProductSpec();
var products = await context.Products
    .Where(spec.ToExpression())
    .ToListAsync();
```

---

## 요약

| 패턴 | 목적 | EF Core 적용 |
|------|------|-------------|
| Aggregate Root | 도메인 무결성 | 네비게이션 속성 캡슐화 |
| Value Object | 불변 값 | OwnsOne/Complex Type |
| Domain Events | 느슨한 결합 | SaveChanges 후 발행 |
| Specification | 쿼리 재사용 | Expression 기반 |
| Clean Architecture | 관심사 분리 | 계층별 프로젝트 분리 |
