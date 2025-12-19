# Chapter 41-48: 고급 주제 심화

## 개요

EF Core의 심화 고급 기능들을 알아봅니다. 인터셉터, 이벤트, 성능 모니터링, 마이크로서비스 패턴 등을 다룹니다.

---

## 41. 인터셉터(Interceptors)

### 41.1 SaveChanges 인터셉터

```csharp
public class AuditInterceptor : SaveChangesInterceptor
{
    private readonly ICurrentUserService _currentUser;
    private readonly TimeProvider _timeProvider;

    public AuditInterceptor(ICurrentUserService currentUser, TimeProvider timeProvider)
    {
        _currentUser = currentUser;
        _timeProvider = timeProvider;
    }

    public override ValueTask<InterceptionResult<int>> SavingChangesAsync(
        DbContextEventData eventData,
        InterceptionResult<int> result,
        CancellationToken cancellationToken = default)
    {
        var context = eventData.Context;
        if (context == null) return ValueTask.FromResult(result);

        var now = _timeProvider.GetUtcNow().UtcDateTime;
        var userId = _currentUser.UserId;

        foreach (var entry in context.ChangeTracker.Entries<IAuditableEntity>())
        {
            switch (entry.State)
            {
                case EntityState.Added:
                    entry.Entity.CreatedAt = now;
                    entry.Entity.CreatedBy = userId;
                    break;

                case EntityState.Modified:
                    entry.Entity.ModifiedAt = now;
                    entry.Entity.ModifiedBy = userId;
                    break;
            }
        }

        return ValueTask.FromResult(result);
    }

    public override async ValueTask<int> SavedChangesAsync(
        SaveChangesCompletedEventData eventData,
        int result,
        CancellationToken cancellationToken = default)
    {
        // 저장 완료 후 로직 (예: 이벤트 발행)
        var context = eventData.Context;
        if (context == null) return result;

        var domainEvents = context.ChangeTracker.Entries<IHasDomainEvents>()
            .SelectMany(e => e.Entity.DomainEvents)
            .ToList();

        foreach (var domainEvent in domainEvents)
        {
            await _mediator.Publish(domainEvent, cancellationToken);
        }

        return result;
    }
}

// 등록
services.AddDbContext<ApplicationDbContext>((sp, options) =>
{
    options.UseSqlServer(connectionString)
           .AddInterceptors(sp.GetRequiredService<AuditInterceptor>());
});
```

### 41.2 Command 인터셉터

```csharp
public class SlowQueryInterceptor : DbCommandInterceptor
{
    private readonly ILogger<SlowQueryInterceptor> _logger;
    private readonly TimeSpan _threshold = TimeSpan.FromSeconds(1);

    public SlowQueryInterceptor(ILogger<SlowQueryInterceptor> logger)
    {
        _logger = logger;
    }

    public override async ValueTask<DbDataReader> ReaderExecutedAsync(
        DbCommand command,
        CommandExecutedEventData eventData,
        DbDataReader result,
        CancellationToken cancellationToken = default)
    {
        if (eventData.Duration > _threshold)
        {
            _logger.LogWarning(
                "Slow query detected ({Duration}ms): {CommandText}",
                eventData.Duration.TotalMilliseconds,
                command.CommandText);
        }

        return result;
    }

    public override DbCommand CommandCreated(
        CommandEndEventData eventData,
        DbCommand result)
    {
        // 쿼리 힌트 추가
        if (result.CommandText.Contains("Products"))
        {
            result.CommandText = result.CommandText.Replace(
                "FROM [Products]",
                "FROM [Products] WITH (NOLOCK)");
        }

        return result;
    }
}
```

### 41.3 Connection 인터셉터

```csharp
public class ConnectionInterceptor : DbConnectionInterceptor
{
    private readonly ILogger<ConnectionInterceptor> _logger;

    public override async ValueTask<InterceptionResult> ConnectionOpeningAsync(
        DbConnection connection,
        ConnectionEventData eventData,
        InterceptionResult result,
        CancellationToken cancellationToken = default)
    {
        _logger.LogDebug("Opening connection to {Database}", connection.Database);
        return result;
    }

    public override async Task ConnectionOpenedAsync(
        DbConnection connection,
        ConnectionEndEventData eventData,
        CancellationToken cancellationToken = default)
    {
        // 연결 후 세션 설정
        await using var command = connection.CreateCommand();
        command.CommandText = "SET TRANSACTION ISOLATION LEVEL READ COMMITTED";
        await command.ExecuteNonQueryAsync(cancellationToken);
    }

    public override async ValueTask<InterceptionResult> ConnectionClosingAsync(
        DbConnection connection,
        ConnectionEventData eventData,
        InterceptionResult result)
    {
        _logger.LogDebug("Closing connection after {Duration}ms", eventData.Duration?.TotalMilliseconds);
        return result;
    }
}
```

---

## 42. 이벤트와 진단

### 42.1 DiagnosticListener

```csharp
public class EfCoreDiagnosticObserver : IObserver<DiagnosticListener>
{
    private readonly ILogger _logger;

    public void OnNext(DiagnosticListener listener)
    {
        if (listener.Name == DbLoggerCategory.Name)
        {
            listener.Subscribe(new EfCoreEventObserver(_logger));
        }
    }

    public void OnCompleted() { }
    public void OnError(Exception error) { }
}

public class EfCoreEventObserver : IObserver<KeyValuePair<string, object?>>
{
    private readonly ILogger _logger;

    public void OnNext(KeyValuePair<string, object?> pair)
    {
        switch (pair.Key)
        {
            case "Microsoft.EntityFrameworkCore.Database.Command.CommandExecuting":
                var commandData = (CommandEventData)pair.Value!;
                _logger.LogDebug("Executing: {Sql}", commandData.Command.CommandText);
                break;

            case "Microsoft.EntityFrameworkCore.Database.Command.CommandExecuted":
                var executedData = (CommandExecutedEventData)pair.Value!;
                _logger.LogDebug("Executed in {Duration}ms", executedData.Duration.TotalMilliseconds);
                break;

            case "Microsoft.EntityFrameworkCore.Query.QueryCompilationStarting":
                _logger.LogDebug("Query compilation starting");
                break;
        }
    }

    public void OnCompleted() { }
    public void OnError(Exception error) { }
}

// 등록
DiagnosticListener.AllListeners.Subscribe(new EfCoreDiagnosticObserver());
```

### 42.2 ChangeTracker 이벤트

```csharp
public class ApplicationDbContext : DbContext
{
    public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options)
        : base(options)
    {
        ChangeTracker.Tracked += OnEntityTracked;
        ChangeTracker.StateChanged += OnEntityStateChanged;
    }

    private void OnEntityTracked(object? sender, EntityTrackedEventArgs e)
    {
        if (!e.FromQuery && e.Entry.State == EntityState.Added)
        {
            Console.WriteLine($"Entity added: {e.Entry.Entity.GetType().Name}");
        }
    }

    private void OnEntityStateChanged(object? sender, EntityStateChangedEventArgs e)
    {
        Console.WriteLine($"State changed: {e.Entry.Entity.GetType().Name} " +
                         $"from {e.OldState} to {e.NewState}");

        // 삭제 감지
        if (e.NewState == EntityState.Deleted)
        {
            // Soft delete로 변환
            if (e.Entry.Entity is ISoftDeletable softDeletable)
            {
                softDeletable.IsDeleted = true;
                softDeletable.DeletedAt = DateTime.UtcNow;
                e.Entry.State = EntityState.Modified;
            }
        }
    }
}
```

---

## 43. 성능 모니터링

### 43.1 Application Insights 통합

```csharp
// NuGet: Microsoft.ApplicationInsights.AspNetCore

services.AddApplicationInsightsTelemetry();

services.AddDbContext<ApplicationDbContext>(options =>
{
    options.UseSqlServer(connectionString)
           .LogTo(message => telemetryClient.TrackTrace(message),
                  LogLevel.Information);
});

// 커스텀 메트릭
public class QueryMetricsInterceptor : DbCommandInterceptor
{
    private readonly TelemetryClient _telemetryClient;

    public override async ValueTask<DbDataReader> ReaderExecutedAsync(
        DbCommand command,
        CommandExecutedEventData eventData,
        DbDataReader result,
        CancellationToken cancellationToken = default)
    {
        var metric = new MetricTelemetry
        {
            Name = "EFCore.Query.Duration",
            Sum = eventData.Duration.TotalMilliseconds
        };

        metric.Properties["CommandType"] = command.CommandType.ToString();
        metric.Properties["Database"] = command.Connection?.Database;

        _telemetryClient.TrackMetric(metric);

        return result;
    }
}
```

### 43.2 OpenTelemetry 통합

```csharp
// NuGet: OpenTelemetry.Instrumentation.EntityFrameworkCore

builder.Services.AddOpenTelemetry()
    .WithTracing(tracing =>
    {
        tracing
            .AddAspNetCoreInstrumentation()
            .AddEntityFrameworkCoreInstrumentation(options =>
            {
                options.SetDbStatementForText = true;
                options.SetDbStatementForStoredProcedure = true;
            })
            .AddSqlClientInstrumentation(options =>
            {
                options.RecordException = true;
                options.SetDbStatementForText = true;
            })
            .AddOtlpExporter();
    })
    .WithMetrics(metrics =>
    {
        metrics
            .AddAspNetCoreInstrumentation()
            .AddRuntimeInstrumentation()
            .AddOtlpExporter();
    });
```

### 43.3 커스텀 성능 카운터

```csharp
public class EfCoreMetrics
{
    private readonly Counter<long> _queryCount;
    private readonly Histogram<double> _queryDuration;
    private readonly UpDownCounter<int> _activeConnections;

    public EfCoreMetrics(IMeterFactory meterFactory)
    {
        var meter = meterFactory.Create("EFCore.Metrics");

        _queryCount = meter.CreateCounter<long>(
            "efcore.queries.total",
            description: "Total number of queries executed");

        _queryDuration = meter.CreateHistogram<double>(
            "efcore.query.duration",
            unit: "ms",
            description: "Query execution duration");

        _activeConnections = meter.CreateUpDownCounter<int>(
            "efcore.connections.active",
            description: "Number of active connections");
    }

    public void RecordQuery(string queryType, double durationMs)
    {
        _queryCount.Add(1, new KeyValuePair<string, object?>("type", queryType));
        _queryDuration.Record(durationMs, new KeyValuePair<string, object?>("type", queryType));
    }

    public void ConnectionOpened() => _activeConnections.Add(1);
    public void ConnectionClosed() => _activeConnections.Add(-1);
}
```

---

## 44. 멀티 테넌시

### 44.1 스키마 기반 멀티 테넌시

```csharp
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

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // 테넌트별 스키마 설정
        modelBuilder.HasDefaultSchema(_tenantService.Schema);

        modelBuilder.Entity<Product>().ToTable("Products", _tenantService.Schema);
        modelBuilder.Entity<Order>().ToTable("Orders", _tenantService.Schema);
    }
}

// 테넌트 서비스
public class TenantService : ITenantService
{
    private readonly IHttpContextAccessor _httpContextAccessor;

    public string TenantId => GetTenantId();
    public string Schema => $"tenant_{TenantId}";

    private string GetTenantId()
    {
        // 헤더, 도메인, 또는 클레임에서 추출
        return _httpContextAccessor.HttpContext?.Request.Headers["X-Tenant-ID"]
            ?? throw new InvalidOperationException("Tenant not identified");
    }
}
```

### 44.2 행 수준 멀티 테넌시

```csharp
public interface ITenantEntity
{
    string TenantId { get; set; }
}

public class Product : ITenantEntity
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public string TenantId { get; set; } = string.Empty;
}

public class TenantDbContext : DbContext
{
    private readonly ITenantService _tenantService;

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // 글로벌 필터로 테넌트 격리
        modelBuilder.Entity<Product>()
            .HasQueryFilter(p => p.TenantId == _tenantService.TenantId);

        modelBuilder.Entity<Order>()
            .HasQueryFilter(o => o.TenantId == _tenantService.TenantId);
    }

    public override Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
    {
        // 새 엔티티에 TenantId 자동 설정
        foreach (var entry in ChangeTracker.Entries<ITenantEntity>()
            .Where(e => e.State == EntityState.Added))
        {
            entry.Entity.TenantId = _tenantService.TenantId;
        }

        return base.SaveChangesAsync(cancellationToken);
    }
}
```

### 44.3 데이터베이스 기반 멀티 테넌시

```csharp
public class TenantConnectionFactory
{
    private readonly ITenantService _tenantService;
    private readonly IConfiguration _configuration;

    public string GetConnectionString()
    {
        var tenant = _tenantService.CurrentTenant;
        return _configuration.GetConnectionString($"Tenant_{tenant.Id}")
            ?? throw new InvalidOperationException($"Connection not configured for tenant {tenant.Id}");
    }
}

// DbContext 팩토리 사용
public class TenantDbContextFactory : IDbContextFactory<ApplicationDbContext>
{
    private readonly TenantConnectionFactory _connectionFactory;
    private readonly IServiceProvider _serviceProvider;

    public ApplicationDbContext CreateDbContext()
    {
        var connectionString = _connectionFactory.GetConnectionString();

        var optionsBuilder = new DbContextOptionsBuilder<ApplicationDbContext>();
        optionsBuilder.UseSqlServer(connectionString);

        return new ApplicationDbContext(optionsBuilder.Options);
    }
}
```

---

## 45. 도메인 이벤트

### 45.1 도메인 이벤트 패턴

```csharp
public interface IDomainEvent
{
    DateTime OccurredOn { get; }
}

public interface IHasDomainEvents
{
    IReadOnlyCollection<IDomainEvent> DomainEvents { get; }
    void ClearDomainEvents();
}

public abstract class Entity : IHasDomainEvents
{
    private readonly List<IDomainEvent> _domainEvents = new();

    public IReadOnlyCollection<IDomainEvent> DomainEvents => _domainEvents.AsReadOnly();

    protected void AddDomainEvent(IDomainEvent domainEvent)
    {
        _domainEvents.Add(domainEvent);
    }

    public void ClearDomainEvents()
    {
        _domainEvents.Clear();
    }
}

// 도메인 이벤트 예시
public record OrderCreatedEvent(int OrderId, int CustomerId, decimal Total) : IDomainEvent
{
    public DateTime OccurredOn { get; } = DateTime.UtcNow;
}

public record OrderStatusChangedEvent(int OrderId, OrderStatus OldStatus, OrderStatus NewStatus) : IDomainEvent
{
    public DateTime OccurredOn { get; } = DateTime.UtcNow;
}
```

### 45.2 이벤트 디스패처

```csharp
public class DomainEventDispatcher : SaveChangesInterceptor
{
    private readonly IMediator _mediator;

    public DomainEventDispatcher(IMediator mediator)
    {
        _mediator = mediator;
    }

    public override async ValueTask<int> SavedChangesAsync(
        SaveChangesCompletedEventData eventData,
        int result,
        CancellationToken cancellationToken = default)
    {
        var context = eventData.Context;
        if (context == null) return result;

        var entities = context.ChangeTracker
            .Entries<IHasDomainEvents>()
            .Where(e => e.Entity.DomainEvents.Any())
            .Select(e => e.Entity)
            .ToList();

        var domainEvents = entities
            .SelectMany(e => e.DomainEvents)
            .ToList();

        // 이벤트 클리어
        foreach (var entity in entities)
        {
            entity.ClearDomainEvents();
        }

        // 이벤트 발행
        foreach (var domainEvent in domainEvents)
        {
            await _mediator.Publish(domainEvent, cancellationToken);
        }

        return result;
    }
}

// 이벤트 핸들러
public class OrderCreatedEventHandler : INotificationHandler<OrderCreatedEvent>
{
    private readonly IEmailService _emailService;
    private readonly IInventoryService _inventoryService;

    public async Task Handle(OrderCreatedEvent notification, CancellationToken cancellationToken)
    {
        // 이메일 발송
        await _emailService.SendOrderConfirmationAsync(notification.OrderId);

        // 재고 예약
        await _inventoryService.ReserveStockAsync(notification.OrderId);
    }
}
```

---

## 46. 분산 트랜잭션 패턴

### 46.1 Outbox 패턴

```csharp
public class OutboxMessage
{
    public Guid Id { get; set; }
    public string Type { get; set; } = string.Empty;
    public string Payload { get; set; } = string.Empty;
    public DateTime CreatedAt { get; set; }
    public DateTime? ProcessedAt { get; set; }
    public int RetryCount { get; set; }
}

public class OutboxDbContext : DbContext
{
    public DbSet<OutboxMessage> OutboxMessages => Set<OutboxMessage>();
}

// 메시지 저장 (트랜잭션 내에서)
public class OrderService
{
    public async Task CreateOrderAsync(Order order)
    {
        await using var transaction = await _context.Database.BeginTransactionAsync();

        try
        {
            _context.Orders.Add(order);

            // Outbox에 이벤트 저장 (같은 트랜잭션)
            var outboxMessage = new OutboxMessage
            {
                Id = Guid.NewGuid(),
                Type = nameof(OrderCreatedEvent),
                Payload = JsonSerializer.Serialize(new OrderCreatedEvent(order.Id, order.CustomerId, order.Total)),
                CreatedAt = DateTime.UtcNow
            };
            _context.OutboxMessages.Add(outboxMessage);

            await _context.SaveChangesAsync();
            await transaction.CommitAsync();
        }
        catch
        {
            await transaction.RollbackAsync();
            throw;
        }
    }
}

// 백그라운드 워커로 메시지 발행
public class OutboxProcessor : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            await using var context = await _contextFactory.CreateDbContextAsync();

            var messages = await context.OutboxMessages
                .Where(m => m.ProcessedAt == null)
                .OrderBy(m => m.CreatedAt)
                .Take(100)
                .ToListAsync(stoppingToken);

            foreach (var message in messages)
            {
                try
                {
                    await _messageBus.PublishAsync(message.Type, message.Payload);
                    message.ProcessedAt = DateTime.UtcNow;
                }
                catch
                {
                    message.RetryCount++;
                }
            }

            await context.SaveChangesAsync(stoppingToken);
            await Task.Delay(TimeSpan.FromSeconds(5), stoppingToken);
        }
    }
}
```

### 46.2 Saga 패턴

```csharp
public class OrderSaga
{
    private readonly OrderSagaState _state;

    public async Task<OrderResult> ExecuteAsync(CreateOrderRequest request)
    {
        try
        {
            // Step 1: 재고 예약
            _state.InventoryReserved = await _inventoryService.ReserveAsync(request.Items);
            if (!_state.InventoryReserved)
                return OrderResult.Failed("Insufficient stock");

            // Step 2: 결제 처리
            _state.PaymentProcessed = await _paymentService.ProcessAsync(request.Payment);
            if (!_state.PaymentProcessed)
            {
                await CompensateAsync();
                return OrderResult.Failed("Payment failed");
            }

            // Step 3: 주문 생성
            _state.OrderCreated = await _orderService.CreateAsync(request);

            // Step 4: 재고 확정
            await _inventoryService.ConfirmReservationAsync(_state.ReservationId);

            return OrderResult.Success(_state.OrderId);
        }
        catch (Exception ex)
        {
            await CompensateAsync();
            throw;
        }
    }

    private async Task CompensateAsync()
    {
        // 역순으로 보상 트랜잭션 실행
        if (_state.PaymentProcessed)
            await _paymentService.RefundAsync(_state.PaymentId);

        if (_state.InventoryReserved)
            await _inventoryService.ReleaseReservationAsync(_state.ReservationId);
    }
}
```

---

## 47. 고급 쿼리 기법

### 47.1 동적 쿼리 빌더

```csharp
public class ProductQueryBuilder
{
    private IQueryable<Product> _query;

    public ProductQueryBuilder(IQueryable<Product> query)
    {
        _query = query;
    }

    public ProductQueryBuilder WithCategory(int? categoryId)
    {
        if (categoryId.HasValue)
            _query = _query.Where(p => p.CategoryId == categoryId.Value);
        return this;
    }

    public ProductQueryBuilder WithPriceRange(decimal? minPrice, decimal? maxPrice)
    {
        if (minPrice.HasValue)
            _query = _query.Where(p => p.Price >= minPrice.Value);
        if (maxPrice.HasValue)
            _query = _query.Where(p => p.Price <= maxPrice.Value);
        return this;
    }

    public ProductQueryBuilder WithSearch(string? searchTerm)
    {
        if (!string.IsNullOrEmpty(searchTerm))
            _query = _query.Where(p => p.Name.Contains(searchTerm));
        return this;
    }

    public ProductQueryBuilder OrderBy(string? sortBy, bool descending = false)
    {
        _query = sortBy?.ToLower() switch
        {
            "name" => descending ? _query.OrderByDescending(p => p.Name) : _query.OrderBy(p => p.Name),
            "price" => descending ? _query.OrderByDescending(p => p.Price) : _query.OrderBy(p => p.Price),
            "created" => descending ? _query.OrderByDescending(p => p.CreatedAt) : _query.OrderBy(p => p.CreatedAt),
            _ => _query.OrderBy(p => p.Id)
        };
        return this;
    }

    public IQueryable<Product> Build() => _query;
}

// 사용
var query = new ProductQueryBuilder(context.Products)
    .WithCategory(request.CategoryId)
    .WithPriceRange(request.MinPrice, request.MaxPrice)
    .WithSearch(request.SearchTerm)
    .OrderBy(request.SortBy, request.Descending)
    .Build();
```

### 47.2 Specification 패턴

```csharp
public interface ISpecification<T>
{
    Expression<Func<T, bool>> Criteria { get; }
    List<Expression<Func<T, object>>> Includes { get; }
    List<string> IncludeStrings { get; }
}

public abstract class Specification<T> : ISpecification<T>
{
    public Expression<Func<T, bool>> Criteria { get; protected set; } = _ => true;
    public List<Expression<Func<T, object>>> Includes { get; } = new();
    public List<string> IncludeStrings { get; } = new();

    protected void AddInclude(Expression<Func<T, object>> includeExpression)
    {
        Includes.Add(includeExpression);
    }
}

public class ActiveProductsSpec : Specification<Product>
{
    public ActiveProductsSpec()
    {
        Criteria = p => p.IsActive && p.StockQuantity > 0;
    }
}

public class ProductsByCategorySpec : Specification<Product>
{
    public ProductsByCategorySpec(int categoryId)
    {
        Criteria = p => p.CategoryId == categoryId;
        AddInclude(p => p.Category);
    }
}

// Repository에서 사용
public async Task<IReadOnlyList<T>> ListAsync<T>(ISpecification<T> spec) where T : class
{
    return await ApplySpecification(spec).ToListAsync();
}

private IQueryable<T> ApplySpecification<T>(ISpecification<T> spec) where T : class
{
    var query = _context.Set<T>().AsQueryable();

    query = query.Where(spec.Criteria);

    query = spec.Includes.Aggregate(query, (current, include) => current.Include(include));
    query = spec.IncludeStrings.Aggregate(query, (current, include) => current.Include(include));

    return query;
}
```

---

## 48. EF Core 최신 기능 (EF Core 8+)

### 48.1 Complex Types

```csharp
// EF Core 8+: Complex Types (Value Objects without identity)
[ComplexType]
public class Money
{
    public decimal Amount { get; set; }
    public string Currency { get; set; } = "USD";
}

public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public Money Price { get; set; } = new();  // Complex Type
}

// 자동으로 Price_Amount, Price_Currency 컬럼 생성
```

### 48.2 Primitive Collections

```csharp
// EF Core 8+: 기본 타입 컬렉션 매핑
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public List<string> Tags { get; set; } = new();  // JSON 배열로 저장
    public List<int> RelatedProductIds { get; set; } = new();
}

// 쿼리
var products = await context.Products
    .Where(p => p.Tags.Contains("electronics"))
    .ToListAsync();
```

### 48.3 Raw SQL for Unmapped Types

```csharp
// EF Core 8+: 매핑되지 않은 타입에 Raw SQL 사용
var statistics = await context.Database
    .SqlQuery<ProductStatistics>($"""
        SELECT
            CategoryId,
            COUNT(*) as ProductCount,
            AVG(Price) as AveragePrice,
            SUM(StockQuantity) as TotalStock
        FROM Products
        GROUP BY CategoryId
        """)
    .ToListAsync();

public class ProductStatistics
{
    public int CategoryId { get; set; }
    public int ProductCount { get; set; }
    public decimal AveragePrice { get; set; }
    public int TotalStock { get; set; }
}
```

### 48.4 DateOnly & TimeOnly

```csharp
// EF Core 8+: DateOnly, TimeOnly 네이티브 지원
public class Event
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public DateOnly EventDate { get; set; }
    public TimeOnly StartTime { get; set; }
    public TimeOnly EndTime { get; set; }
}

// 쿼리
var todayEvents = await context.Events
    .Where(e => e.EventDate == DateOnly.FromDateTime(DateTime.Today))
    .Where(e => e.StartTime >= new TimeOnly(9, 0))
    .ToListAsync();
```

### 48.5 HierarchyId (SQL Server)

```csharp
// EF Core 8+: SQL Server HierarchyId 지원
public class Employee
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public HierarchyId Path { get; set; } = null!;
}

// 쿼리: 모든 하위 직원 찾기
var manager = await context.Employees.FindAsync(managerId);
var subordinates = await context.Employees
    .Where(e => e.Path.IsDescendantOf(manager!.Path))
    .ToListAsync();
```

---

## 요약

| 기능 | 용도 | 복잡도 |
|------|------|--------|
| Interceptors | 쿼리/저장 가로채기 | 중간 |
| DiagnosticListener | 세밀한 모니터링 | 높음 |
| 멀티 테넌시 | 테넌트 격리 | 높음 |
| 도메인 이벤트 | DDD 이벤트 패턴 | 중간 |
| Outbox 패턴 | 분산 트랜잭션 | 높음 |
| Specification | 동적 쿼리 | 중간 |
| EF Core 8+ | 최신 기능 | 낮음~중간 |

---

## 학습 완료

이 문서 시리즈를 통해 EF Core의 기초부터 고급 주제까지 학습했습니다:

1. **기초**: DbContext, 엔티티, 관계
2. **데이터 모델링**: Fluent API, 상속, Value Objects
3. **쿼리**: LINQ, 로딩 전략, 필터
4. **데이터 조작**: Change Tracking, 저장, 동시성
5. **마이그레이션**: 기본 및 고급 기법
6. **성능**: 최적화, 벌크 작업, 풀링
7. **테스트**: 단위/통합 테스트
8. **패턴**: Repository, DDD, Clean Architecture
9. **고급**: DB별 기능, JSON, Database-First
10. **실습**: 블로그, E-Commerce 프로젝트
11. **모범 사례**: 수명 관리, 보안, 구조화
12. **흔한 실수**: N+1, 추적 문제, Lazy Loading
13. **심화**: 인터셉터, 이벤트, 분산 패턴
