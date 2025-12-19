# Chapter 03: 모델 정의 기초

## 개요

EF Core에서 엔티티 클래스를 작성하고 데이터베이스 스키마와 매핑하는 방법을 알아봅니다. Data Annotations와 Fluent API의 차이점과 사용법을 상세히 다룹니다.

---

## 3.1 엔티티 클래스 작성

### 간략 설명
엔티티 클래스는 데이터베이스 테이블과 매핑되는 C# 클래스입니다. POCO(Plain Old CLR Object) 원칙을 따라 간단하게 작성합니다.

### 상세 설명

#### 기본 엔티티 클래스

```csharp
// 가장 기본적인 엔티티
public class Product
{
    public int Id { get; set; }           // 기본 키 (컨벤션)
    public string Name { get; set; }       // 필수 속성
    public decimal Price { get; set; }     // 값 타입
    public string? Description { get; set; } // nullable 속성
}
```

#### 컨벤션 기반 매핑

EF Core는 다음 규칙을 자동으로 적용합니다:

```csharp
public class Customer
{
    // 1. Id 또는 {클래스명}Id → 기본 키로 인식
    public int Id { get; set; }
    // 또는
    public int CustomerId { get; set; }

    // 2. string → nvarchar(max)
    public string Name { get; set; }

    // 3. string? → nullable 컬럼
    public string? MiddleName { get; set; }

    // 4. DateTime → datetime2(7)
    public DateTime CreatedAt { get; set; }

    // 5. decimal → decimal(18,2)
    public decimal Balance { get; set; }

    // 6. bool → bit
    public bool IsActive { get; set; }

    // 7. enum → int
    public CustomerType Type { get; set; }

    // 8. byte[] → varbinary(max)
    public byte[]? ProfileImage { get; set; }

    // 9. Guid → uniqueidentifier
    public Guid ExternalId { get; set; }
}

public enum CustomerType
{
    Regular = 0,
    Premium = 1,
    VIP = 2
}
```

#### 생성되는 SQL (SQL Server)

```sql
CREATE TABLE [Customers] (
    [Id] int NOT NULL IDENTITY,
    [Name] nvarchar(max) NOT NULL,
    [MiddleName] nvarchar(max) NULL,
    [CreatedAt] datetime2 NOT NULL,
    [Balance] decimal(18,2) NOT NULL,
    [IsActive] bit NOT NULL,
    [Type] int NOT NULL,
    [ProfileImage] varbinary(max) NULL,
    [ExternalId] uniqueidentifier NOT NULL,
    CONSTRAINT [PK_Customers] PRIMARY KEY ([Id])
);
```

#### 권장 엔티티 작성 패턴

```csharp
// ✅ 권장되는 엔티티 작성 방식
public class Order
{
    // 기본 키
    public int Id { get; private set; }

    // 필수 속성들 - null이 아님을 보장
    public string OrderNumber { get; private set; } = string.Empty;
    public DateTime OrderDate { get; private set; }
    public OrderStatus Status { get; private set; }

    // 선택적 속성들 - nullable
    public string? Notes { get; set; }
    public DateTime? ShippedDate { get; private set; }

    // 계산된 속성 - 데이터베이스에 저장되지 않음
    public decimal TotalAmount => OrderItems.Sum(i => i.Quantity * i.UnitPrice);

    // 외래 키
    public int CustomerId { get; private set; }

    // 네비게이션 속성
    public Customer Customer { get; private set; } = null!;
    public ICollection<OrderItem> OrderItems { get; private set; } = new List<OrderItem>();

    // EF Core용 private 생성자
    private Order() { }

    // 팩토리 메서드
    public static Order Create(int customerId, string orderNumber)
    {
        return new Order
        {
            CustomerId = customerId,
            OrderNumber = orderNumber,
            OrderDate = DateTime.UtcNow,
            Status = OrderStatus.Pending
        };
    }

    // 도메인 메서드
    public void Ship()
    {
        if (Status != OrderStatus.Confirmed)
            throw new InvalidOperationException("주문이 확정되지 않았습니다.");

        Status = OrderStatus.Shipped;
        ShippedDate = DateTime.UtcNow;
    }
}

public enum OrderStatus
{
    Pending,
    Confirmed,
    Shipped,
    Delivered,
    Cancelled
}
```

#### 엔티티 설계 주의사항

```csharp
// ❌ 피해야 할 패턴

public class BadEntity
{
    public int ID { get; set; }  // ID 대문자 - 일관성 없음

    public string name;  // 필드 사용 - 속성이어야 함

    public string Data { get; }  // getter만 있음 - EF Core가 값 설정 불가

    public List<Item> Items { get; set; }  // 구체 타입 - ICollection 권장

    // 정적 속성 - 매핑되지 않음
    public static int Counter { get; set; }
}

// ✅ 올바른 패턴

public class GoodEntity
{
    public int Id { get; set; }  // PascalCase, 일관된 명명

    public string Name { get; set; } = string.Empty;  // 속성 사용

    public string Data { get; private set; } = string.Empty;  // private set 허용

    public ICollection<Item> Items { get; set; } = new List<Item>();  // 인터페이스 타입
}
```

---

## 3.2 Data Annotations vs Fluent API

### 간략 설명
EF Core는 두 가지 방식으로 모델을 구성할 수 있습니다. Data Annotations는 어트리뷰트를 사용하고, Fluent API는 코드로 구성합니다.

### Data Annotations

```csharp
using System.ComponentModel.DataAnnotations;
using System.ComponentModel.DataAnnotations.Schema;

[Table("Products", Schema = "catalog")]
[Index(nameof(Sku), IsUnique = true)]
public class Product
{
    [Key]
    [DatabaseGenerated(DatabaseGeneratedOption.Identity)]
    public int Id { get; set; }

    [Required]
    [StringLength(100)]
    public string Name { get; set; } = string.Empty;

    [Required]
    [StringLength(50)]
    [Column("SKU")]
    public string Sku { get; set; } = string.Empty;

    [Column(TypeName = "decimal(18,4)")]
    [Range(0, 1000000)]
    public decimal Price { get; set; }

    [MaxLength(2000)]
    public string? Description { get; set; }

    [Required]
    [Url]
    public string ImageUrl { get; set; } = string.Empty;

    [NotMapped]  // 데이터베이스에 매핑하지 않음
    public string DisplayName => $"{Name} ({Sku})";

    [Timestamp]
    public byte[] RowVersion { get; set; } = null!;

    // 외래 키
    [ForeignKey(nameof(Category))]
    public int CategoryId { get; set; }

    public Category Category { get; set; } = null!;
}
```

### Fluent API

```csharp
public class CatalogContext : DbContext
{
    public DbSet<Product> Products => Set<Product>();
    public DbSet<Category> Categories => Set<Category>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Product>(entity =>
        {
            // 테이블 설정
            entity.ToTable("Products", "catalog");

            // 기본 키
            entity.HasKey(p => p.Id);
            entity.Property(p => p.Id)
                .UseIdentityColumn();

            // 속성 설정
            entity.Property(p => p.Name)
                .IsRequired()
                .HasMaxLength(100);

            entity.Property(p => p.Sku)
                .IsRequired()
                .HasMaxLength(50)
                .HasColumnName("SKU");

            entity.Property(p => p.Price)
                .HasColumnType("decimal(18,4)");

            entity.Property(p => p.Description)
                .HasMaxLength(2000);

            entity.Property(p => p.ImageUrl)
                .IsRequired();

            // 인덱스
            entity.HasIndex(p => p.Sku)
                .IsUnique();

            // 무시할 속성
            entity.Ignore(p => p.DisplayName);

            // 동시성 토큰
            entity.Property(p => p.RowVersion)
                .IsRowVersion();

            // 관계 설정
            entity.HasOne(p => p.Category)
                .WithMany(c => c.Products)
                .HasForeignKey(p => p.CategoryId)
                .OnDelete(DeleteBehavior.Restrict);
        });
    }
}
```

### 비교 표

| 기능 | Data Annotations | Fluent API |
|------|-----------------|------------|
| 가독성 | 엔티티 클래스에서 바로 확인 | 별도 구성 필요 |
| 유연성 | 제한적 | 매우 유연 |
| 관계 설정 | 기본적인 것만 | 모든 옵션 지원 |
| 유효성 검사 | 자동 통합 | 별도 구현 필요 |
| 분리 | 도메인과 혼재 | 완전 분리 가능 |
| 고급 기능 | 일부 미지원 | 전체 지원 |

### 권장 사용 패턴

```csharp
// ✅ 권장: Data Annotations (간단한 유효성 검사)
public class User
{
    public int Id { get; set; }

    [Required]
    [StringLength(100)]
    public string Name { get; set; } = string.Empty;

    [Required]
    [EmailAddress]
    public string Email { get; set; } = string.Empty;
}

// ✅ 권장: Fluent API (데이터베이스 스키마 구성)
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<User>(entity =>
    {
        entity.ToTable("Users");
        entity.HasIndex(u => u.Email).IsUnique();
        entity.Property(u => u.Email).HasMaxLength(256);
    });
}

// ⚠️ 혼용 시 Fluent API가 우선
```

### IEntityTypeConfiguration으로 구성 분리

```csharp
// Configurations/ProductConfiguration.cs
public class ProductConfiguration : IEntityTypeConfiguration<Product>
{
    public void Configure(EntityTypeBuilder<Product> builder)
    {
        builder.ToTable("Products", "catalog");

        builder.HasKey(p => p.Id);

        builder.Property(p => p.Name)
            .IsRequired()
            .HasMaxLength(100);

        builder.Property(p => p.Sku)
            .IsRequired()
            .HasMaxLength(50);

        builder.HasIndex(p => p.Sku)
            .IsUnique();

        builder.HasOne(p => p.Category)
            .WithMany(c => c.Products)
            .HasForeignKey(p => p.CategoryId);
    }
}

// DbContext에서 적용
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    // 개별 적용
    modelBuilder.ApplyConfiguration(new ProductConfiguration());

    // 또는 어셈블리의 모든 구성 자동 적용
    modelBuilder.ApplyConfigurationsFromAssembly(typeof(CatalogContext).Assembly);
}
```

---

## 3.3 기본 키(Primary Key) 설정

### 간략 설명
기본 키는 엔티티를 고유하게 식별하는 속성입니다. EF Core는 다양한 기본 키 생성 전략을 지원합니다.

### 기본 키 유형

#### 1. 자동 증가 정수 키

```csharp
// 가장 일반적인 방식
public class Blog
{
    public int Id { get; set; }  // 자동으로 IDENTITY로 설정
}

// Fluent API로 명시적 설정
modelBuilder.Entity<Blog>()
    .Property(b => b.Id)
    .UseIdentityColumn(seed: 1, increment: 1);  // SQL Server
    // .UseSerialColumn();  // PostgreSQL
    // .ValueGeneratedOnAdd();  // 일반적인 설정
```

#### 2. GUID 키

```csharp
public class Order
{
    public Guid Id { get; set; }  // 클라이언트에서 생성 가능
}

// 서버에서 생성하도록 설정
modelBuilder.Entity<Order>()
    .Property(o => o.Id)
    .HasDefaultValueSql("NEWSEQUENTIALID()");  // SQL Server (순차적 GUID)
    // .HasDefaultValueSql("gen_random_uuid()");  // PostgreSQL

// 클라이언트에서 생성
public class Order
{
    public Guid Id { get; set; } = Guid.NewGuid();
}
```

#### 3. 복합 키

```csharp
public class OrderItem
{
    public int OrderId { get; set; }
    public int ProductId { get; set; }
    public int Quantity { get; set; }

    public Order Order { get; set; } = null!;
    public Product Product { get; set; } = null!;
}

// Fluent API (Data Annotations로 불가능)
modelBuilder.Entity<OrderItem>()
    .HasKey(oi => new { oi.OrderId, oi.ProductId });
```

#### 4. 대체 키 (Alternate Key)

```csharp
public class User
{
    public int Id { get; set; }
    public string Username { get; set; } = string.Empty;  // 고유해야 함
    public string Email { get; set; } = string.Empty;      // 고유해야 함
}

modelBuilder.Entity<User>(entity =>
{
    entity.HasKey(u => u.Id);

    // 대체 키 - 외래 키로 참조 가능
    entity.HasAlternateKey(u => u.Username);
    entity.HasAlternateKey(u => u.Email);
});

// 대체 키를 외래 키로 사용
public class Comment
{
    public int Id { get; set; }
    public string AuthorUsername { get; set; } = string.Empty;

    public User Author { get; set; } = null!;
}

modelBuilder.Entity<Comment>()
    .HasOne(c => c.Author)
    .WithMany()
    .HasForeignKey(c => c.AuthorUsername)
    .HasPrincipalKey(u => u.Username);  // 대체 키 참조
```

### 키 생성 전략 비교

| 전략 | 장점 | 단점 | 사용 상황 |
|------|------|------|----------|
| Identity (int) | 간단, 작은 크기, 빠른 조인 | 분산 환경 부적합, 예측 가능 | 대부분의 상황 |
| GUID | 분산 환경 적합, 예측 불가 | 크기(16바이트), 인덱스 성능 | 분산 시스템, 보안 |
| Sequential GUID | GUID 장점 + 인덱스 성능 | DB 종속적 | 분산 + 성능 필요 |
| Hi/Lo | 라운드트립 감소 | 구현 복잡 | 대량 삽입 |
| 수동 지정 | 완전한 제어 | 충돌 관리 필요 | 특수한 경우 |

### Hi/Lo 알고리즘

```csharp
// 데이터베이스 시퀀스 사용
modelBuilder.Entity<Order>()
    .Property(o => o.Id)
    .UseHiLo("orders_hilo_seq");

// 동작 방식:
// 1. 시작 시 DB에서 블록(예: 1-10) 할당받음
// 2. 메모리에서 1, 2, 3, ... 9, 10 사용
// 3. 소진되면 다음 블록(11-20) 할당
// 장점: SaveChanges 전에 ID를 알 수 있음
```

---

## 3.4 속성(Property) 구성

### 간략 설명
엔티티의 각 속성을 데이터베이스 컬럼에 매핑하고, 다양한 옵션을 설정하는 방법을 알아봅니다.

### 기본 속성 구성

```csharp
modelBuilder.Entity<Product>(entity =>
{
    // 컬럼 이름
    entity.Property(p => p.Name)
        .HasColumnName("ProductName");

    // 컬럼 타입
    entity.Property(p => p.Price)
        .HasColumnType("money");  // SQL Server money 타입

    // 길이 제한
    entity.Property(p => p.Sku)
        .HasMaxLength(50);

    // 필수 여부
    entity.Property(p => p.Name)
        .IsRequired();

    // 유니코드
    entity.Property(p => p.Code)
        .IsUnicode(false);  // varchar 사용 (nvarchar 대신)

    // 정밀도 (decimal, DateTime)
    entity.Property(p => p.Price)
        .HasPrecision(18, 4);

    entity.Property(p => p.ExactTime)
        .HasPrecision(3);  // datetime2(3)

    // 기본값
    entity.Property(p => p.IsActive)
        .HasDefaultValue(true);

    entity.Property(p => p.CreatedAt)
        .HasDefaultValueSql("GETUTCDATE()");

    // 컬럼 순서
    entity.Property(p => p.Id).HasColumnOrder(0);
    entity.Property(p => p.Name).HasColumnOrder(1);
});
```

### 값 변환 (Value Conversion)

```csharp
// 1. 열거형을 문자열로 저장
public class Order
{
    public int Id { get; set; }
    public OrderStatus Status { get; set; }
}

modelBuilder.Entity<Order>()
    .Property(o => o.Status)
    .HasConversion<string>();  // int 대신 문자열로 저장

// 2. 커스텀 변환기
modelBuilder.Entity<User>()
    .Property(u => u.Email)
    .HasConversion(
        v => v.ToLower(),           // 저장 시 소문자로
        v => v);                     // 읽기 시 그대로

// 3. 복잡한 변환기
public class MoneyConverter : ValueConverter<Money, decimal>
{
    public MoneyConverter()
        : base(
            money => money.Amount,           // Money → decimal
            value => new Money(value))       // decimal → Money
    {
    }
}

modelBuilder.Entity<Product>()
    .Property(p => p.Price)
    .HasConversion(new MoneyConverter());

// 4. JSON 변환 (EF Core 7+)
public class Product
{
    public int Id { get; set; }
    public Dictionary<string, string> Metadata { get; set; } = new();
}

modelBuilder.Entity<Product>()
    .Property(p => p.Metadata)
    .HasConversion(
        v => JsonSerializer.Serialize(v, (JsonSerializerOptions?)null),
        v => JsonSerializer.Deserialize<Dictionary<string, string>>(v, (JsonSerializerOptions?)null)!);

// EF Core 7+ JSON 컬럼 (권장)
modelBuilder.Entity<Product>()
    .OwnsOne(p => p.Metadata, builder =>
    {
        builder.ToJson();
    });
```

### Shadow Property (그림자 속성)

```csharp
// 엔티티 클래스에 없지만 데이터베이스에 존재하는 속성
public class Blog
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    // LastModified는 클래스에 없음
}

modelBuilder.Entity<Blog>()
    .Property<DateTime>("LastModified");

// 사용 방법
context.Entry(blog).Property("LastModified").CurrentValue = DateTime.UtcNow;

// 쿼리에서 사용
var blogs = context.Blogs
    .OrderBy(b => EF.Property<DateTime>(b, "LastModified"))
    .ToList();
```

### 계산 컬럼

```csharp
public class OrderItem
{
    public int Id { get; set; }
    public int Quantity { get; set; }
    public decimal UnitPrice { get; set; }
    public decimal TotalPrice { get; set; }  // 계산 컬럼
}

modelBuilder.Entity<OrderItem>()
    .Property(oi => oi.TotalPrice)
    .HasComputedColumnSql("[Quantity] * [UnitPrice]");

// 저장된 계산 컬럼 (성능 향상)
modelBuilder.Entity<OrderItem>()
    .Property(oi => oi.TotalPrice)
    .HasComputedColumnSql("[Quantity] * [UnitPrice]", stored: true);
```

### 속성 접근 모드

```csharp
public class User
{
    private string _email = string.Empty;

    public int Id { get; set; }

    // 백킹 필드 사용
    public string Email
    {
        get => _email;
        set => _email = value?.ToLower() ?? string.Empty;
    }

    // 읽기 전용 속성
    public string DisplayEmail => _email;
}

modelBuilder.Entity<User>()
    .Property(u => u.Email)
    .HasField("_email")
    .UsePropertyAccessMode(PropertyAccessMode.Field);  // 항상 필드 사용

// PropertyAccessMode 옵션:
// - Property: 항상 속성 사용 (기본값)
// - Field: 항상 필드 사용
// - FieldDuringConstruction: 생성 시 필드, 이후 속성
// - PreferField: 가능하면 필드
// - PreferProperty: 가능하면 속성
```

---

## 실습 예제

### 종합 엔티티 모델 예제

```csharp
// Entities/Category.cs
public class Category
{
    public int Id { get; set; }

    [Required]
    [StringLength(100)]
    public string Name { get; set; } = string.Empty;

    public string? Description { get; set; }

    public int? ParentCategoryId { get; set; }
    public Category? ParentCategory { get; set; }
    public ICollection<Category> SubCategories { get; set; } = new List<Category>();
    public ICollection<Product> Products { get; set; } = new List<Product>();
}

// Entities/Product.cs
public class Product
{
    public int Id { get; set; }

    [Required]
    [StringLength(200)]
    public string Name { get; set; } = string.Empty;

    [Required]
    [StringLength(50)]
    public string Sku { get; set; } = string.Empty;

    public decimal Price { get; set; }
    public int StockQuantity { get; set; }
    public bool IsActive { get; set; } = true;

    public DateTime CreatedAt { get; set; }
    public DateTime? UpdatedAt { get; set; }

    public int CategoryId { get; set; }
    public Category Category { get; set; } = null!;

    public ICollection<OrderItem> OrderItems { get; set; } = new List<OrderItem>();
}

// Configurations/CategoryConfiguration.cs
public class CategoryConfiguration : IEntityTypeConfiguration<Category>
{
    public void Configure(EntityTypeBuilder<Category> builder)
    {
        builder.ToTable("Categories");

        builder.HasKey(c => c.Id);

        builder.Property(c => c.Name)
            .IsRequired()
            .HasMaxLength(100);

        builder.Property(c => c.Description)
            .HasMaxLength(500);

        // 자기 참조 관계
        builder.HasOne(c => c.ParentCategory)
            .WithMany(c => c.SubCategories)
            .HasForeignKey(c => c.ParentCategoryId)
            .OnDelete(DeleteBehavior.Restrict);

        // 인덱스
        builder.HasIndex(c => c.Name);
    }
}

// Configurations/ProductConfiguration.cs
public class ProductConfiguration : IEntityTypeConfiguration<Product>
{
    public void Configure(EntityTypeBuilder<Product> builder)
    {
        builder.ToTable("Products");

        builder.HasKey(p => p.Id);

        builder.Property(p => p.Name)
            .IsRequired()
            .HasMaxLength(200);

        builder.Property(p => p.Sku)
            .IsRequired()
            .HasMaxLength(50);

        builder.Property(p => p.Price)
            .HasPrecision(18, 2);

        builder.Property(p => p.IsActive)
            .HasDefaultValue(true);

        builder.Property(p => p.CreatedAt)
            .HasDefaultValueSql("GETUTCDATE()");

        // 유니크 인덱스
        builder.HasIndex(p => p.Sku)
            .IsUnique();

        // 복합 인덱스
        builder.HasIndex(p => new { p.CategoryId, p.IsActive });

        // 관계
        builder.HasOne(p => p.Category)
            .WithMany(c => c.Products)
            .HasForeignKey(p => p.CategoryId)
            .OnDelete(DeleteBehavior.Cascade);
    }
}

// DbContext
public class StoreContext : DbContext
{
    public StoreContext(DbContextOptions<StoreContext> options)
        : base(options)
    {
    }

    public DbSet<Category> Categories => Set<Category>();
    public DbSet<Product> Products => Set<Product>();
    public DbSet<Order> Orders => Set<Order>();
    public DbSet<OrderItem> OrderItems => Set<OrderItem>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(StoreContext).Assembly);
    }
}
```

---

## 요약

| 항목 | 핵심 내용 |
|------|----------|
| 엔티티 클래스 | POCO 원칙, 컨벤션 기반 매핑 |
| Data Annotations | 간단한 구성, 유효성 검사 통합 |
| Fluent API | 유연한 구성, 전체 기능 지원 |
| 기본 키 | Identity, GUID, 복합 키 지원 |
| 속성 구성 | 타입, 길이, 기본값, 변환 등 |
| 구성 분리 | IEntityTypeConfiguration 권장 |

## 다음 장 예고

다음 Part에서는 엔티티 간의 관계(일대다, 일대일, 다대다)를 설정하는 방법을 상세히 알아봅니다.
