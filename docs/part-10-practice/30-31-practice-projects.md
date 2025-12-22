# Chapter 30-31: 실습 프로젝트

## 개요

실제 프로젝트를 통해 EF Core의 모든 개념을 종합적으로 적용해봅니다. 블로그 시스템과 E-Commerce 시스템을 단계별로 구현합니다.

---

## 30. 블로그 시스템 구현

### 30.1 도메인 모델 설계

```csharp
// 엔티티 정의
public class Blog
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public string Url { get; set; } = string.Empty;
    public DateTime CreatedAt { get; set; }
    public bool IsActive { get; set; } = true;

    // 관계
    public int OwnerId { get; set; }
    public User Owner { get; set; } = null!;
    public ICollection<Post> Posts { get; set; } = new List<Post>();
}

public class Post
{
    public int Id { get; set; }
    public string Title { get; set; } = string.Empty;
    public string Content { get; set; } = string.Empty;
    public string? Summary { get; set; }
    public DateTime PublishedAt { get; set; }
    public DateTime? ModifiedAt { get; set; }
    public PostStatus Status { get; set; } = PostStatus.Draft;
    public int ViewCount { get; set; }

    // 관계
    public int BlogId { get; set; }
    public Blog Blog { get; set; } = null!;
    public int AuthorId { get; set; }
    public User Author { get; set; } = null!;
    public ICollection<Comment> Comments { get; set; } = new List<Comment>();
    public ICollection<Tag> Tags { get; set; } = new List<Tag>();
}

public class Comment
{
    public int Id { get; set; }
    public string Content { get; set; } = string.Empty;
    public DateTime CreatedAt { get; set; }
    public bool IsApproved { get; set; }

    // Self-referencing (대댓글)
    public int? ParentCommentId { get; set; }
    public Comment? ParentComment { get; set; }
    public ICollection<Comment> Replies { get; set; } = new List<Comment>();

    // 관계
    public int PostId { get; set; }
    public Post Post { get; set; } = null!;
    public int AuthorId { get; set; }
    public User Author { get; set; } = null!;
}

public class Tag
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public string Slug { get; set; } = string.Empty;

    // Many-to-Many
    public ICollection<Post> Posts { get; set; } = new List<Post>();
}

public class User
{
    public int Id { get; set; }
    public string Username { get; set; } = string.Empty;
    public string Email { get; set; } = string.Empty;
    public string PasswordHash { get; set; } = string.Empty;
    public UserProfile Profile { get; set; } = null!;

    public ICollection<Blog> Blogs { get; set; } = new List<Blog>();
    public ICollection<Post> Posts { get; set; } = new List<Post>();
    public ICollection<Comment> Comments { get; set; } = new List<Comment>();
}

public class UserProfile
{
    public int Id { get; set; }
    public string? DisplayName { get; set; }
    public string? Bio { get; set; }
    public string? AvatarUrl { get; set; }

    public int UserId { get; set; }
    public User User { get; set; } = null!;
}

public enum PostStatus
{
    Draft,
    Published,
    Archived
}
```

### 31.2 DbContext 구성

```csharp
public class BlogDbContext : DbContext
{
    public BlogDbContext(DbContextOptions<BlogDbContext> options)
        : base(options)
    {
    }

    public DbSet<Blog> Blogs => Set<Blog>();
    public DbSet<Post> Posts => Set<Post>();
    public DbSet<Comment> Comments => Set<Comment>();
    public DbSet<Tag> Tags => Set<Tag>();
    public DbSet<User> Users => Set<User>();
    public DbSet<UserProfile> UserProfiles => Set<UserProfile>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(BlogDbContext).Assembly);
    }

    public override Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
    {
        // 감사 로그 자동 설정
        var now = DateTime.UtcNow;

        foreach (var entry in ChangeTracker.Entries<Post>())
        {
            if (entry.State == EntityState.Modified)
            {
                entry.Entity.ModifiedAt = now;
            }
        }

        return base.SaveChangesAsync(cancellationToken);
    }
}

// 엔티티 설정
public class PostConfiguration : IEntityTypeConfiguration<Post>
{
    public void Configure(EntityTypeBuilder<Post> builder)
    {
        builder.HasKey(p => p.Id);

        builder.Property(p => p.Title)
            .IsRequired()
            .HasMaxLength(200);

        builder.Property(p => p.Content)
            .IsRequired();

        builder.Property(p => p.Summary)
            .HasMaxLength(500);

        // 인덱스
        builder.HasIndex(p => p.PublishedAt);
        builder.HasIndex(p => new { p.BlogId, p.Status });

        // 관계
        builder.HasOne(p => p.Blog)
            .WithMany(b => b.Posts)
            .HasForeignKey(p => p.BlogId)
            .OnDelete(DeleteBehavior.Cascade);

        builder.HasOne(p => p.Author)
            .WithMany(u => u.Posts)
            .HasForeignKey(p => p.AuthorId)
            .OnDelete(DeleteBehavior.Restrict);

        // Many-to-Many
        builder.HasMany(p => p.Tags)
            .WithMany(t => t.Posts)
            .UsingEntity(j => j.ToTable("PostTags"));

        // 글로벌 필터 (Published만)
        builder.HasQueryFilter(p => p.Status == PostStatus.Published);
    }
}

public class CommentConfiguration : IEntityTypeConfiguration<Comment>
{
    public void Configure(EntityTypeBuilder<Comment> builder)
    {
        builder.HasKey(c => c.Id);

        builder.Property(c => c.Content)
            .IsRequired()
            .HasMaxLength(2000);

        // Self-referencing
        builder.HasOne(c => c.ParentComment)
            .WithMany(c => c.Replies)
            .HasForeignKey(c => c.ParentCommentId)
            .OnDelete(DeleteBehavior.Restrict);

        // 글로벌 필터 (승인된 댓글만)
        builder.HasQueryFilter(c => c.IsApproved);
    }
}
```

### 30.3 Repository 패턴 구현

```csharp
public interface IPostRepository
{
    Task<Post?> GetByIdAsync(int id, CancellationToken ct = default);
    Task<Post?> GetByIdWithDetailsAsync(int id, CancellationToken ct = default);
    Task<IReadOnlyList<Post>> GetRecentPostsAsync(int count, CancellationToken ct = default);
    Task<IReadOnlyList<Post>> GetByTagAsync(string tagSlug, int page, int pageSize, CancellationToken ct = default);
    Task<IReadOnlyList<Post>> SearchAsync(string query, CancellationToken ct = default);
    Task AddAsync(Post post, CancellationToken ct = default);
    void Update(Post post);
    void Delete(Post post);
}

public class PostRepository : IPostRepository
{
    private readonly BlogDbContext _context;

    public PostRepository(BlogDbContext context)
    {
        _context = context;
    }

    public async Task<Post?> GetByIdAsync(int id, CancellationToken ct = default)
    {
        return await _context.Posts.FindAsync(new object[] { id }, ct);
    }

    public async Task<Post?> GetByIdWithDetailsAsync(int id, CancellationToken ct = default)
    {
        return await _context.Posts
            .Include(p => p.Author)
                .ThenInclude(a => a.Profile)
            .Include(p => p.Tags)
            .Include(p => p.Comments.Where(c => c.ParentCommentId == null))
                .ThenInclude(c => c.Author)
            .AsSplitQuery()
            .FirstOrDefaultAsync(p => p.Id == id, ct);
    }

    public async Task<IReadOnlyList<Post>> GetRecentPostsAsync(int count, CancellationToken ct = default)
    {
        return await _context.Posts
            .Include(p => p.Author)
            .Include(p => p.Tags)
            .OrderByDescending(p => p.PublishedAt)
            .Take(count)
            .ToListAsync(ct);
    }

    public async Task<IReadOnlyList<Post>> GetByTagAsync(
        string tagSlug, int page, int pageSize, CancellationToken ct = default)
    {
        return await _context.Posts
            .Include(p => p.Author)
            .Include(p => p.Tags)
            .Where(p => p.Tags.Any(t => t.Slug == tagSlug))
            .OrderByDescending(p => p.PublishedAt)
            .Skip((page - 1) * pageSize)
            .Take(pageSize)
            .ToListAsync(ct);
    }

    public async Task<IReadOnlyList<Post>> SearchAsync(string query, CancellationToken ct = default)
    {
        var searchTerms = query.ToLower().Split(' ', StringSplitOptions.RemoveEmptyEntries);

        return await _context.Posts
            .Include(p => p.Author)
            .Where(p => searchTerms.All(term =>
                p.Title.ToLower().Contains(term) ||
                p.Content.ToLower().Contains(term)))
            .OrderByDescending(p => p.PublishedAt)
            .Take(50)
            .ToListAsync(ct);
    }

    public async Task AddAsync(Post post, CancellationToken ct = default)
    {
        await _context.Posts.AddAsync(post, ct);
    }

    public void Update(Post post)
    {
        _context.Posts.Update(post);
    }

    public void Delete(Post post)
    {
        _context.Posts.Remove(post);
    }
}
```

### 30.4 서비스 레이어

```csharp
public interface IPostService
{
    Task<PostDto?> GetPostAsync(int id);
    Task<PagedResult<PostSummaryDto>> GetPostsAsync(PostQueryParams queryParams);
    Task<int> CreatePostAsync(CreatePostRequest request);
    Task UpdatePostAsync(int id, UpdatePostRequest request);
    Task DeletePostAsync(int id);
    Task IncrementViewCountAsync(int id);
}

public class PostService : IPostService
{
    private readonly IPostRepository _postRepository;
    private readonly IUnitOfWork _unitOfWork;
    private readonly ICurrentUserService _currentUser;

    public PostService(
        IPostRepository postRepository,
        IUnitOfWork unitOfWork,
        ICurrentUserService currentUser)
    {
        _postRepository = postRepository;
        _unitOfWork = unitOfWork;
        _currentUser = currentUser;
    }

    public async Task<PostDto?> GetPostAsync(int id)
    {
        var post = await _postRepository.GetByIdWithDetailsAsync(id);
        if (post == null) return null;

        // 조회수 증가 (비동기, fire-and-forget)
        _ = IncrementViewCountAsync(id);

        return MapToDto(post);
    }

    public async Task<int> CreatePostAsync(CreatePostRequest request)
    {
        var post = new Post
        {
            Title = request.Title,
            Content = request.Content,
            Summary = GenerateSummary(request.Content),
            BlogId = request.BlogId,
            AuthorId = _currentUser.UserId,
            PublishedAt = request.PublishNow ? DateTime.UtcNow : default,
            Status = request.PublishNow ? PostStatus.Published : PostStatus.Draft
        };

        // 태그 처리
        if (request.TagIds?.Any() == true)
        {
            var tags = await _unitOfWork.Tags
                .GetByIdsAsync(request.TagIds);
            post.Tags = tags.ToList();
        }

        await _postRepository.AddAsync(post);
        await _unitOfWork.SaveChangesAsync();

        return post.Id;
    }

    public async Task IncrementViewCountAsync(int id)
    {
        // ExecuteUpdate로 효율적인 업데이트
        await _unitOfWork.Context.Posts
            .Where(p => p.Id == id)
            .ExecuteUpdateAsync(s => s
                .SetProperty(p => p.ViewCount, p => p.ViewCount + 1));
    }

    private static string GenerateSummary(string content, int maxLength = 300)
    {
        if (string.IsNullOrEmpty(content)) return string.Empty;

        var plainText = StripHtml(content);
        return plainText.Length <= maxLength
            ? plainText
            : plainText[..maxLength] + "...";
    }

    private static PostDto MapToDto(Post post) => new()
    {
        Id = post.Id,
        Title = post.Title,
        Content = post.Content,
        Summary = post.Summary,
        PublishedAt = post.PublishedAt,
        ViewCount = post.ViewCount,
        Author = new AuthorDto
        {
            Id = post.Author.Id,
            Username = post.Author.Username,
            DisplayName = post.Author.Profile?.DisplayName,
            AvatarUrl = post.Author.Profile?.AvatarUrl
        },
        Tags = post.Tags.Select(t => new TagDto
        {
            Id = t.Id,
            Name = t.Name,
            Slug = t.Slug
        }).ToList(),
        CommentCount = post.Comments.Count
    };
}
```

---

## 31. E-Commerce 시스템 구현

### 31.1 도메인 모델

```csharp
// Aggregate Root: Order
public class Order
{
    public int Id { get; set; }
    public string OrderNumber { get; set; } = string.Empty;
    public DateTime OrderDate { get; set; }
    public OrderStatus Status { get; set; }
    public decimal SubTotal { get; set; }
    public decimal Tax { get; set; }
    public decimal ShippingCost { get; set; }
    public decimal Total { get; set; }

    // Value Object
    public Address ShippingAddress { get; set; } = null!;
    public Address BillingAddress { get; set; } = null!;

    // 관계
    public int CustomerId { get; set; }
    public Customer Customer { get; set; } = null!;
    public ICollection<OrderItem> Items { get; set; } = new List<OrderItem>();
    public ICollection<OrderStatusHistory> StatusHistory { get; set; } = new List<OrderStatusHistory>();

    // 도메인 메서드
    public void AddItem(Product product, int quantity)
    {
        var existingItem = Items.FirstOrDefault(i => i.ProductId == product.Id);

        if (existingItem != null)
        {
            existingItem.Quantity += quantity;
        }
        else
        {
            Items.Add(new OrderItem
            {
                ProductId = product.Id,
                ProductName = product.Name,
                UnitPrice = product.Price,
                Quantity = quantity
            });
        }

        RecalculateTotals();
    }

    public void UpdateStatus(OrderStatus newStatus, string? note = null)
    {
        if (!CanTransitionTo(newStatus))
            throw new InvalidOperationException($"Cannot transition from {Status} to {newStatus}");

        Status = newStatus;
        StatusHistory.Add(new OrderStatusHistory
        {
            Status = newStatus,
            ChangedAt = DateTime.UtcNow,
            Note = note
        });
    }

    private void RecalculateTotals()
    {
        SubTotal = Items.Sum(i => i.LineTotal);
        Tax = SubTotal * 0.1m;  // 10% tax
        Total = SubTotal + Tax + ShippingCost;
    }

    private bool CanTransitionTo(OrderStatus newStatus) => (Status, newStatus) switch
    {
        (OrderStatus.Pending, OrderStatus.Confirmed) => true,
        (OrderStatus.Pending, OrderStatus.Cancelled) => true,
        (OrderStatus.Confirmed, OrderStatus.Processing) => true,
        (OrderStatus.Confirmed, OrderStatus.Cancelled) => true,
        (OrderStatus.Processing, OrderStatus.Shipped) => true,
        (OrderStatus.Shipped, OrderStatus.Delivered) => true,
        (OrderStatus.Delivered, OrderStatus.Returned) => true,
        _ => false
    };
}

public class OrderItem
{
    public int Id { get; set; }
    public int OrderId { get; set; }
    public int ProductId { get; set; }
    public string ProductName { get; set; } = string.Empty;
    public decimal UnitPrice { get; set; }
    public int Quantity { get; set; }
    public decimal Discount { get; set; }

    public decimal LineTotal => (UnitPrice * Quantity) - Discount;

    public Order Order { get; set; } = null!;
    public Product Product { get; set; } = null!;
}

// Value Object
[Owned]
public class Address
{
    public string Street { get; set; } = string.Empty;
    public string City { get; set; } = string.Empty;
    public string State { get; set; } = string.Empty;
    public string PostalCode { get; set; } = string.Empty;
    public string Country { get; set; } = string.Empty;
}

public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public string? Description { get; set; }
    public string Sku { get; set; } = string.Empty;
    public decimal Price { get; set; }
    public decimal? CompareAtPrice { get; set; }
    public int StockQuantity { get; set; }
    public bool IsActive { get; set; } = true;

    // JSON 컬럼
    public ProductAttributes? Attributes { get; set; }

    // 관계
    public int CategoryId { get; set; }
    public Category Category { get; set; } = null!;
    public ICollection<ProductImage> Images { get; set; } = new List<ProductImage>();

    public bool IsInStock => StockQuantity > 0;
    public bool IsOnSale => CompareAtPrice.HasValue && CompareAtPrice > Price;
}

public class ProductAttributes
{
    public string? Color { get; set; }
    public string? Size { get; set; }
    public string? Material { get; set; }
    public decimal? Weight { get; set; }
    public Dictionary<string, string> CustomAttributes { get; set; } = new();
}

public class Customer
{
    public int Id { get; set; }
    public string Email { get; set; } = string.Empty;
    public string FirstName { get; set; } = string.Empty;
    public string LastName { get; set; } = string.Empty;
    public string? Phone { get; set; }
    public DateTime RegisteredAt { get; set; }
    public CustomerTier Tier { get; set; } = CustomerTier.Standard;

    public Address? DefaultShippingAddress { get; set; }
    public Address? DefaultBillingAddress { get; set; }

    public ICollection<Order> Orders { get; set; } = new List<Order>();

    public string FullName => $"{FirstName} {LastName}";
}

public enum OrderStatus
{
    Pending,
    Confirmed,
    Processing,
    Shipped,
    Delivered,
    Cancelled,
    Returned
}

public enum CustomerTier
{
    Standard,
    Silver,
    Gold,
    Platinum
}
```

### 31.2 DbContext 및 설정

```csharp
public class ECommerceDbContext : DbContext
{
    public ECommerceDbContext(DbContextOptions<ECommerceDbContext> options)
        : base(options)
    {
    }

    public DbSet<Product> Products => Set<Product>();
    public DbSet<Category> Categories => Set<Category>();
    public DbSet<Customer> Customers => Set<Customer>();
    public DbSet<Order> Orders => Set<Order>();
    public DbSet<OrderItem> OrderItems => Set<OrderItem>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Product 설정
        modelBuilder.Entity<Product>(entity =>
        {
            entity.HasKey(p => p.Id);

            entity.Property(p => p.Name).IsRequired().HasMaxLength(200);
            entity.Property(p => p.Sku).IsRequired().HasMaxLength(50);
            entity.Property(p => p.Price).HasPrecision(18, 2);
            entity.Property(p => p.CompareAtPrice).HasPrecision(18, 2);

            entity.HasIndex(p => p.Sku).IsUnique();
            entity.HasIndex(p => p.CategoryId);
            entity.HasIndex(p => new { p.IsActive, p.StockQuantity });

            // JSON 컬럼
            entity.OwnsOne(p => p.Attributes, attr =>
            {
                attr.ToJson();
            });

            // 글로벌 필터
            entity.HasQueryFilter(p => p.IsActive);
        });

        // Order 설정
        modelBuilder.Entity<Order>(entity =>
        {
            entity.HasKey(o => o.Id);

            entity.Property(o => o.OrderNumber)
                .IsRequired()
                .HasMaxLength(20);

            entity.Property(o => o.SubTotal).HasPrecision(18, 2);
            entity.Property(o => o.Tax).HasPrecision(18, 2);
            entity.Property(o => o.ShippingCost).HasPrecision(18, 2);
            entity.Property(o => o.Total).HasPrecision(18, 2);

            entity.HasIndex(o => o.OrderNumber).IsUnique();
            entity.HasIndex(o => o.CustomerId);
            entity.HasIndex(o => o.OrderDate);

            // Owned Types (Value Objects)
            entity.OwnsOne(o => o.ShippingAddress, addr =>
            {
                addr.Property(a => a.Street).HasMaxLength(200);
                addr.Property(a => a.City).HasMaxLength(100);
                addr.Property(a => a.State).HasMaxLength(100);
                addr.Property(a => a.PostalCode).HasMaxLength(20);
                addr.Property(a => a.Country).HasMaxLength(100);
            });

            entity.OwnsOne(o => o.BillingAddress, addr =>
            {
                addr.Property(a => a.Street).HasMaxLength(200);
                addr.Property(a => a.City).HasMaxLength(100);
                addr.Property(a => a.State).HasMaxLength(100);
                addr.Property(a => a.PostalCode).HasMaxLength(20);
                addr.Property(a => a.Country).HasMaxLength(100);
            });

            // 관계
            entity.HasOne(o => o.Customer)
                .WithMany(c => c.Orders)
                .HasForeignKey(o => o.CustomerId)
                .OnDelete(DeleteBehavior.Restrict);
        });

        // OrderItem 설정
        modelBuilder.Entity<OrderItem>(entity =>
        {
            entity.HasKey(oi => oi.Id);

            entity.Property(oi => oi.ProductName).IsRequired().HasMaxLength(200);
            entity.Property(oi => oi.UnitPrice).HasPrecision(18, 2);
            entity.Property(oi => oi.Discount).HasPrecision(18, 2);

            // LineTotal은 계산 컬럼 (선택적)
            entity.Ignore(oi => oi.LineTotal);

            entity.HasOne(oi => oi.Order)
                .WithMany(o => o.Items)
                .HasForeignKey(oi => oi.OrderId)
                .OnDelete(DeleteBehavior.Cascade);

            entity.HasOne(oi => oi.Product)
                .WithMany()
                .HasForeignKey(oi => oi.ProductId)
                .OnDelete(DeleteBehavior.Restrict);
        });

        // Customer 설정
        modelBuilder.Entity<Customer>(entity =>
        {
            entity.HasKey(c => c.Id);

            entity.Property(c => c.Email).IsRequired().HasMaxLength(255);
            entity.Property(c => c.FirstName).IsRequired().HasMaxLength(100);
            entity.Property(c => c.LastName).IsRequired().HasMaxLength(100);

            entity.HasIndex(c => c.Email).IsUnique();

            entity.OwnsOne(c => c.DefaultShippingAddress);
            entity.OwnsOne(c => c.DefaultBillingAddress);
        });
    }
}
```

### 31.3 주문 처리 서비스

```csharp
public interface IOrderService
{
    Task<OrderDto> CreateOrderAsync(CreateOrderRequest request);
    Task<OrderDto?> GetOrderAsync(int orderId);
    Task<IReadOnlyList<OrderDto>> GetCustomerOrdersAsync(int customerId);
    Task UpdateOrderStatusAsync(int orderId, OrderStatus newStatus, string? note = null);
    Task<bool> CancelOrderAsync(int orderId, string reason);
}

public class OrderService : IOrderService
{
    private readonly ECommerceDbContext _context;
    private readonly IInventoryService _inventoryService;
    private readonly IOrderNumberGenerator _orderNumberGenerator;
    private readonly IEventPublisher _eventPublisher;

    public OrderService(
        ECommerceDbContext context,
        IInventoryService inventoryService,
        IOrderNumberGenerator orderNumberGenerator,
        IEventPublisher eventPublisher)
    {
        _context = context;
        _inventoryService = inventoryService;
        _orderNumberGenerator = orderNumberGenerator;
        _eventPublisher = eventPublisher;
    }

    public async Task<OrderDto> CreateOrderAsync(CreateOrderRequest request)
    {
        // 실행 전략 사용 (재시도 지원)
        var strategy = _context.Database.CreateExecutionStrategy();

        return await strategy.ExecuteAsync(async () =>
        {
            await using var transaction = await _context.Database.BeginTransactionAsync();

            try
            {
                // 고객 조회
                var customer = await _context.Customers
                    .FirstOrDefaultAsync(c => c.Id == request.CustomerId)
                    ?? throw new NotFoundException("Customer not found");

                // 상품 조회 및 재고 확인
                var productIds = request.Items.Select(i => i.ProductId).ToList();
                var products = await _context.Products
                    .Where(p => productIds.Contains(p.Id))
                    .ToListAsync();

                // 재고 검증
                foreach (var item in request.Items)
                {
                    var product = products.FirstOrDefault(p => p.Id == item.ProductId)
                        ?? throw new NotFoundException($"Product {item.ProductId} not found");

                    if (product.StockQuantity < item.Quantity)
                        throw new InsufficientStockException(product.Name, product.StockQuantity);
                }

                // 주문 생성
                var order = new Order
                {
                    OrderNumber = await _orderNumberGenerator.GenerateAsync(),
                    OrderDate = DateTime.UtcNow,
                    Status = OrderStatus.Pending,
                    CustomerId = request.CustomerId,
                    ShippingAddress = request.ShippingAddress ?? customer.DefaultShippingAddress!,
                    BillingAddress = request.BillingAddress ?? customer.DefaultBillingAddress!,
                    ShippingCost = CalculateShippingCost(request)
                };

                // 주문 아이템 추가
                foreach (var item in request.Items)
                {
                    var product = products.First(p => p.Id == item.ProductId);
                    order.AddItem(product, item.Quantity);
                }

                // 초기 상태 히스토리
                order.StatusHistory.Add(new OrderStatusHistory
                {
                    Status = OrderStatus.Pending,
                    ChangedAt = DateTime.UtcNow,
                    Note = "Order created"
                });

                _context.Orders.Add(order);

                // 재고 감소
                foreach (var item in request.Items)
                {
                    await _inventoryService.DeductStockAsync(item.ProductId, item.Quantity);
                }

                await _context.SaveChangesAsync();
                await transaction.CommitAsync();

                // 도메인 이벤트 발행
                await _eventPublisher.PublishAsync(new OrderCreatedEvent(order.Id, order.OrderNumber));

                return MapToDto(order);
            }
            catch
            {
                await transaction.RollbackAsync();
                throw;
            }
        });
    }

    public async Task<OrderDto?> GetOrderAsync(int orderId)
    {
        var order = await _context.Orders
            .Include(o => o.Customer)
            .Include(o => o.Items)
                .ThenInclude(i => i.Product)
            .Include(o => o.StatusHistory.OrderByDescending(h => h.ChangedAt).Take(5))
            .AsSplitQuery()
            .FirstOrDefaultAsync(o => o.Id == orderId);

        return order == null ? null : MapToDto(order);
    }

    public async Task<IReadOnlyList<OrderDto>> GetCustomerOrdersAsync(int customerId)
    {
        var orders = await _context.Orders
            .Include(o => o.Items)
            .Where(o => o.CustomerId == customerId)
            .OrderByDescending(o => o.OrderDate)
            .Take(50)
            .ToListAsync();

        return orders.Select(MapToDto).ToList();
    }

    public async Task UpdateOrderStatusAsync(int orderId, OrderStatus newStatus, string? note = null)
    {
        var order = await _context.Orders
            .Include(o => o.StatusHistory)
            .FirstOrDefaultAsync(o => o.Id == orderId)
            ?? throw new NotFoundException("Order not found");

        order.UpdateStatus(newStatus, note);

        await _context.SaveChangesAsync();

        await _eventPublisher.PublishAsync(new OrderStatusChangedEvent(orderId, newStatus));
    }

    public async Task<bool> CancelOrderAsync(int orderId, string reason)
    {
        var strategy = _context.Database.CreateExecutionStrategy();

        return await strategy.ExecuteAsync(async () =>
        {
            await using var transaction = await _context.Database.BeginTransactionAsync();

            try
            {
                var order = await _context.Orders
                    .Include(o => o.Items)
                    .Include(o => o.StatusHistory)
                    .FirstOrDefaultAsync(o => o.Id == orderId);

                if (order == null) return false;

                order.UpdateStatus(OrderStatus.Cancelled, reason);

                // 재고 복구
                foreach (var item in order.Items)
                {
                    await _inventoryService.RestoreStockAsync(item.ProductId, item.Quantity);
                }

                await _context.SaveChangesAsync();
                await transaction.CommitAsync();

                await _eventPublisher.PublishAsync(new OrderCancelledEvent(orderId, reason));

                return true;
            }
            catch
            {
                await transaction.RollbackAsync();
                throw;
            }
        });
    }

    private static OrderDto MapToDto(Order order) => new()
    {
        Id = order.Id,
        OrderNumber = order.OrderNumber,
        OrderDate = order.OrderDate,
        Status = order.Status,
        SubTotal = order.SubTotal,
        Tax = order.Tax,
        ShippingCost = order.ShippingCost,
        Total = order.Total,
        Items = order.Items.Select(i => new OrderItemDto
        {
            ProductId = i.ProductId,
            ProductName = i.ProductName,
            UnitPrice = i.UnitPrice,
            Quantity = i.Quantity,
            LineTotal = i.LineTotal
        }).ToList()
    };

    private static decimal CalculateShippingCost(CreateOrderRequest request)
    {
        // 간단한 배송비 계산 로직
        return request.Items.Sum(i => i.Quantity) > 5 ? 0 : 5.99m;
    }
}
```

### 31.4 통합 테스트

```csharp
public class OrderServiceIntegrationTests : IClassFixture<DatabaseFixture>
{
    private readonly DatabaseFixture _fixture;

    public OrderServiceIntegrationTests(DatabaseFixture fixture)
    {
        _fixture = fixture;
    }

    [Fact]
    public async Task CreateOrder_WithValidData_ShouldSucceed()
    {
        // Arrange
        await using var context = _fixture.CreateContext();
        var orderService = CreateOrderService(context);

        var customer = new Customer
        {
            Email = "test@example.com",
            FirstName = "Test",
            LastName = "User",
            RegisteredAt = DateTime.UtcNow,
            DefaultShippingAddress = new Address
            {
                Street = "123 Main St",
                City = "Seoul",
                State = "Seoul",
                PostalCode = "12345",
                Country = "Korea"
            }
        };
        context.Customers.Add(customer);

        var product = new Product
        {
            Name = "Test Product",
            Sku = "TEST-001",
            Price = 100.00m,
            StockQuantity = 10
        };
        context.Products.Add(product);
        await context.SaveChangesAsync();

        var request = new CreateOrderRequest
        {
            CustomerId = customer.Id,
            Items = new[]
            {
                new CreateOrderItemRequest { ProductId = product.Id, Quantity = 2 }
            }
        };

        // Act
        var result = await orderService.CreateOrderAsync(request);

        // Assert
        Assert.NotNull(result);
        Assert.Equal(OrderStatus.Pending, result.Status);
        Assert.Equal(200.00m, result.SubTotal);
        Assert.Single(result.Items);

        // 재고 확인
        var updatedProduct = await context.Products.FindAsync(product.Id);
        Assert.Equal(8, updatedProduct!.StockQuantity);
    }

    [Fact]
    public async Task CreateOrder_WithInsufficientStock_ShouldThrow()
    {
        // Arrange
        await using var context = _fixture.CreateContext();
        var orderService = CreateOrderService(context);

        var customer = new Customer { /* ... */ };
        var product = new Product { StockQuantity = 1 };
        context.Customers.Add(customer);
        context.Products.Add(product);
        await context.SaveChangesAsync();

        var request = new CreateOrderRequest
        {
            CustomerId = customer.Id,
            Items = new[]
            {
                new CreateOrderItemRequest { ProductId = product.Id, Quantity = 5 }
            }
        };

        // Act & Assert
        await Assert.ThrowsAsync<InsufficientStockException>(
            () => orderService.CreateOrderAsync(request));
    }

    [Fact]
    public async Task CancelOrder_ShouldRestoreStock()
    {
        // Arrange
        await using var context = _fixture.CreateContext();
        // ... 주문 생성

        // Act
        var result = await orderService.CancelOrderAsync(orderId, "Customer request");

        // Assert
        Assert.True(result);

        var order = await context.Orders.FindAsync(orderId);
        Assert.Equal(OrderStatus.Cancelled, order!.Status);

        var product = await context.Products.FindAsync(productId);
        Assert.Equal(originalStock, product!.StockQuantity);  // 재고 복구 확인
    }
}
```

---

## 요약

| 프로젝트 | 주요 학습 포인트 |
|---------|----------------|
| 블로그 시스템 | 관계 매핑, 글로벌 필터, Repository 패턴 |
| E-Commerce | DDD, Value Objects, 트랜잭션, 동시성 |

## 다음 Part 예고

다음 Part에서는 EF Core 모범 사례(Best Practices)를 알아봅니다.
