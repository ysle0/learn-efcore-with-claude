# Chapter 05: Fluent API 심화

## 개요

Fluent API를 사용한 고급 모델 구성 기법을 알아봅니다. OnModelCreating 메서드의 구조화, 엔티티 구성 분리, 인덱스와 복합 키 설정 방법을 상세히 다룹니다.

---

## 5.1 OnModelCreating 메서드

### 간략 설명
OnModelCreating은 DbContext에서 모델을 구성하는 핵심 메서드입니다. 이 메서드에서 엔티티, 관계, 인덱스 등을 설정합니다.

### 기본 구조

```csharp
public class ApplicationDbContext : DbContext
{
    public DbSet<Blog> Blogs => Set<Blog>();
    public DbSet<Post> Posts => Set<Post>();

    public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options)
        : base(options)
    {
    }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // 1. 기본 스키마 설정
        modelBuilder.HasDefaultSchema("app");

        // 2. 전역 설정
        ConfigureConventions(modelBuilder);

        // 3. 엔티티별 구성
        ConfigureBlog(modelBuilder);
        ConfigurePost(modelBuilder);

        // 4. 시드 데이터
        SeedData(modelBuilder);

        base.OnModelCreating(modelBuilder);
    }

    private void ConfigureConventions(ModelBuilder modelBuilder)
    {
        // 모든 string 속성에 최대 길이 적용
        foreach (var entity in modelBuilder.Model.GetEntityTypes())
        {
            foreach (var property in entity.GetProperties()
                .Where(p => p.ClrType == typeof(string)))
            {
                if (property.GetMaxLength() == null)
                {
                    property.SetMaxLength(256);
                }
            }
        }
    }

    private void ConfigureBlog(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Blog>(entity =>
        {
            entity.ToTable("Blogs");
            entity.HasKey(b => b.Id);
            entity.Property(b => b.Url).HasMaxLength(500).IsRequired();
        });
    }

    private void ConfigurePost(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Post>(entity =>
        {
            entity.ToTable("Posts");
            entity.HasKey(p => p.Id);

            entity.HasOne(p => p.Blog)
                .WithMany(b => b.Posts)
                .HasForeignKey(p => p.BlogId);
        });
    }

    private void SeedData(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Blog>().HasData(
            new Blog { Id = 1, Url = "https://example.com", Name = "Sample Blog" }
        );
    }
}
```

### ModelBuilder 주요 메서드

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    // 스키마 설정
    modelBuilder.HasDefaultSchema("dbo");

    // 시퀀스 정의
    modelBuilder.HasSequence<int>("OrderNumbers", schema: "shared")
        .StartsAt(1000)
        .IncrementsBy(1);

    // 엔티티 무시
    modelBuilder.Ignore<AuditLog>();

    // 엔티티 접근
    modelBuilder.Entity<Blog>();

    // 모든 엔티티 반복
    foreach (var entityType in modelBuilder.Model.GetEntityTypes())
    {
        Console.WriteLine($"Entity: {entityType.Name}");
    }

    // 특정 타입의 엔티티만 필터링
    var entitiesToAudit = modelBuilder.Model.GetEntityTypes()
        .Where(e => typeof(IAuditable).IsAssignableFrom(e.ClrType));
}
```

---

## 5.2 엔티티 구성 분리 (IEntityTypeConfiguration)

### 간략 설명
엔티티 구성을 별도의 클래스로 분리하여 코드를 깔끔하게 유지하고 재사용성을 높입니다.

### IEntityTypeConfiguration 구현

```csharp
// Configurations/BlogConfiguration.cs
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;

public class BlogConfiguration : IEntityTypeConfiguration<Blog>
{
    public void Configure(EntityTypeBuilder<Blog> builder)
    {
        // 테이블 설정
        builder.ToTable("Blogs", "blogging");

        // 기본 키
        builder.HasKey(b => b.Id);

        // 속성 구성
        builder.Property(b => b.Id)
            .UseIdentityColumn();

        builder.Property(b => b.Name)
            .IsRequired()
            .HasMaxLength(200);

        builder.Property(b => b.Url)
            .IsRequired()
            .HasMaxLength(500);

        builder.Property(b => b.Description)
            .HasMaxLength(2000);

        builder.Property(b => b.CreatedAt)
            .HasDefaultValueSql("GETUTCDATE()");

        // 인덱스
        builder.HasIndex(b => b.Url)
            .IsUnique();

        builder.HasIndex(b => b.Name);

        // 관계
        builder.HasMany(b => b.Posts)
            .WithOne(p => p.Blog)
            .HasForeignKey(p => p.BlogId)
            .OnDelete(DeleteBehavior.Cascade);

        // 시드 데이터
        builder.HasData(
            new Blog
            {
                Id = 1,
                Name = "Tech Blog",
                Url = "https://tech.example.com",
                CreatedAt = new DateTime(2024, 1, 1, 0, 0, 0, DateTimeKind.Utc)
            }
        );
    }
}
```

### 구성 클래스 적용

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    // 방법 1: 개별 적용
    modelBuilder.ApplyConfiguration(new BlogConfiguration());
    modelBuilder.ApplyConfiguration(new PostConfiguration());
    modelBuilder.ApplyConfiguration(new CommentConfiguration());

    // 방법 2: 어셈블리에서 모든 구성 자동 적용 (권장)
    modelBuilder.ApplyConfigurationsFromAssembly(typeof(ApplicationDbContext).Assembly);

    // 방법 3: 특정 어셈블리에서 필터링하여 적용
    modelBuilder.ApplyConfigurationsFromAssembly(
        typeof(ApplicationDbContext).Assembly,
        type => type.Namespace?.Contains("Configurations") == true);
}
```

### 기본 구성 클래스 활용

```csharp
// 공통 구성을 위한 기본 클래스
public abstract class BaseEntityConfiguration<T> : IEntityTypeConfiguration<T>
    where T : BaseEntity
{
    public virtual void Configure(EntityTypeBuilder<T> builder)
    {
        // 공통 속성 구성
        builder.Property(e => e.CreatedAt)
            .HasDefaultValueSql("GETUTCDATE()");

        builder.Property(e => e.UpdatedAt);

        builder.Property(e => e.IsDeleted)
            .HasDefaultValue(false);

        // Soft Delete 필터
        builder.HasQueryFilter(e => !e.IsDeleted);
    }
}

// 기본 엔티티
public abstract class BaseEntity
{
    public int Id { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime? UpdatedAt { get; set; }
    public bool IsDeleted { get; set; }
}

// 구체적인 구성
public class ProductConfiguration : BaseEntityConfiguration<Product>
{
    public override void Configure(EntityTypeBuilder<Product> builder)
    {
        base.Configure(builder);  // 기본 구성 적용

        builder.ToTable("Products");

        builder.Property(p => p.Name)
            .IsRequired()
            .HasMaxLength(200);

        builder.Property(p => p.Price)
            .HasPrecision(18, 2);

        builder.HasIndex(p => p.Sku)
            .IsUnique();
    }
}
```

### 프로젝트 구조

```
DataAccess/
├── DbContext/
│   └── ApplicationDbContext.cs
├── Entities/
│   ├── BaseEntity.cs
│   ├── Blog.cs
│   ├── Post.cs
│   └── Comment.cs
├── Configurations/
│   ├── BaseEntityConfiguration.cs
│   ├── BlogConfiguration.cs
│   ├── PostConfiguration.cs
│   └── CommentConfiguration.cs
└── Extensions/
    └── ModelBuilderExtensions.cs
```

---

## 5.3 인덱스 설정

### 간략 설명
인덱스는 쿼리 성능을 향상시키는 데 필수적입니다. EF Core에서 다양한 유형의 인덱스를 구성하는 방법을 알아봅니다.

### 단일 컬럼 인덱스

```csharp
// Data Annotations
[Index(nameof(Email))]
public class User
{
    public int Id { get; set; }
    public string Email { get; set; } = string.Empty;
}

// Fluent API
modelBuilder.Entity<User>()
    .HasIndex(u => u.Email);
```

### 복합 인덱스

```csharp
// Data Annotations
[Index(nameof(FirstName), nameof(LastName))]
public class Person
{
    public int Id { get; set; }
    public string FirstName { get; set; } = string.Empty;
    public string LastName { get; set; } = string.Empty;
}

// Fluent API
modelBuilder.Entity<Person>()
    .HasIndex(p => new { p.FirstName, p.LastName });
```

### 유니크 인덱스

```csharp
// Data Annotations
[Index(nameof(Email), IsUnique = true)]
public class User
{
    public int Id { get; set; }
    public string Email { get; set; } = string.Empty;
}

// Fluent API
modelBuilder.Entity<User>()
    .HasIndex(u => u.Email)
    .IsUnique();

// 유니크 제약 조건 (대안)
modelBuilder.Entity<User>()
    .HasAlternateKey(u => u.Email);
```

### 인덱스 이름과 필터

```csharp
modelBuilder.Entity<Product>(entity =>
{
    // 인덱스 이름 지정
    entity.HasIndex(p => p.Sku)
        .HasDatabaseName("IX_Products_SKU")
        .IsUnique();

    // 필터링된 인덱스 (부분 인덱스)
    entity.HasIndex(p => p.Email)
        .HasFilter("[Email] IS NOT NULL")
        .HasDatabaseName("IX_Products_Email_NotNull");

    // 삭제되지 않은 항목만 인덱싱
    entity.HasIndex(p => p.Name)
        .HasFilter("[IsDeleted] = 0")
        .HasDatabaseName("IX_Products_Name_Active");
});
```

### 포함 열 (Included Columns)

```csharp
// SQL Server 전용
modelBuilder.Entity<Order>()
    .HasIndex(o => o.CustomerId)
    .IncludeProperties(o => new { o.OrderDate, o.TotalAmount })
    .HasDatabaseName("IX_Orders_CustomerId_Include");

// 생성되는 SQL:
// CREATE INDEX IX_Orders_CustomerId_Include
// ON Orders (CustomerId)
// INCLUDE (OrderDate, TotalAmount)
```

### 내림차순 인덱스

```csharp
// EF Core 7+
modelBuilder.Entity<Post>()
    .HasIndex(p => p.PublishedAt)
    .IsDescending();

// 복합 인덱스에서 혼합
modelBuilder.Entity<Post>()
    .HasIndex(p => new { p.BlogId, p.PublishedAt })
    .IsDescending(false, true);  // BlogId ASC, PublishedAt DESC
```

### 인덱스 전략

```csharp
public class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> builder)
    {
        // 1. 자주 검색되는 컬럼
        builder.HasIndex(o => o.OrderNumber)
            .IsUnique();

        // 2. 외래 키 (자동 생성되지만 명시 가능)
        builder.HasIndex(o => o.CustomerId);

        // 3. 날짜 범위 검색
        builder.HasIndex(o => o.OrderDate);

        // 4. 상태 기반 필터링
        builder.HasIndex(o => o.Status)
            .HasFilter("[Status] IN (0, 1)")  // Pending, Processing
            .HasDatabaseName("IX_Orders_Status_Active");

        // 5. 복합 검색 패턴
        builder.HasIndex(o => new { o.CustomerId, o.OrderDate })
            .IsDescending(false, true)
            .HasDatabaseName("IX_Orders_Customer_Date");
    }
}
```

---

## 5.4 복합 키(Composite Key) 설정

### 간략 설명
복합 키는 두 개 이상의 속성으로 구성된 기본 키입니다. 조인 테이블이나 자연 키가 필요한 경우에 사용됩니다.

### 기본 복합 키

```csharp
// 조인 엔티티
public class StudentCourse
{
    public int StudentId { get; set; }
    public int CourseId { get; set; }
    public DateTime EnrolledAt { get; set; }

    public Student Student { get; set; } = null!;
    public Course Course { get; set; } = null!;
}

// Fluent API (Data Annotations로 불가능)
modelBuilder.Entity<StudentCourse>()
    .HasKey(sc => new { sc.StudentId, sc.CourseId });
```

### 복합 외래 키

```csharp
// 복합 키를 참조하는 관계
public class Region
{
    public string CountryCode { get; set; } = string.Empty;
    public string RegionCode { get; set; } = string.Empty;
    public string Name { get; set; } = string.Empty;

    public ICollection<City> Cities { get; set; } = new List<City>();
}

public class City
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;

    // 복합 외래 키
    public string CountryCode { get; set; } = string.Empty;
    public string RegionCode { get; set; } = string.Empty;

    public Region Region { get; set; } = null!;
}

// Fluent API
modelBuilder.Entity<Region>()
    .HasKey(r => new { r.CountryCode, r.RegionCode });

modelBuilder.Entity<City>()
    .HasOne(c => c.Region)
    .WithMany(r => r.Cities)
    .HasForeignKey(c => new { c.CountryCode, c.RegionCode });
```

### 자연 키 vs 대리 키

```csharp
// 자연 키 (Natural Key) - 비즈니스 의미가 있는 키
public class Country
{
    public string IsoCode { get; set; } = string.Empty;  // "US", "KR" 등
    public string Name { get; set; } = string.Empty;
}

modelBuilder.Entity<Country>()
    .HasKey(c => c.IsoCode);

// 대리 키 (Surrogate Key) - 의미 없는 기술적 키
public class Country
{
    public int Id { get; set; }  // 자동 생성
    public string IsoCode { get; set; } = string.Empty;
    public string Name { get; set; } = string.Empty;
}

modelBuilder.Entity<Country>()
    .HasKey(c => c.Id);

modelBuilder.Entity<Country>()
    .HasAlternateKey(c => c.IsoCode);  // 대체 키로 설정
```

### 복합 키 주의사항

```csharp
// ❌ 문제가 될 수 있는 패턴
public class OrderLine
{
    // 복합 키
    public int OrderId { get; set; }
    public int LineNumber { get; set; }

    public Order Order { get; set; } = null!;
}

// Find 사용 시 주의
var line = await context.OrderLines.FindAsync(orderId, lineNumber);  // 순서 중요!

// ✅ 명확한 조회
var line = await context.OrderLines
    .FirstOrDefaultAsync(ol => ol.OrderId == orderId && ol.LineNumber == lineNumber);
```

---

## 고급 Fluent API 패턴

### 조건부 구성

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    var isDevelopment = Environment.GetEnvironmentVariable("ASPNETCORE_ENVIRONMENT")
        == "Development";

    modelBuilder.Entity<User>(entity =>
    {
        entity.ToTable("Users");

        // 개발 환경에서만 민감한 데이터 로깅
        if (isDevelopment)
        {
            entity.Property(u => u.Password)
                .HasMaxLength(100);
        }
        else
        {
            entity.Property(u => u.Password)
                .HasMaxLength(256);  // 해시된 비밀번호
        }
    });
}
```

### 글로벌 필터

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    // Soft Delete 글로벌 필터
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

// 인터페이스
public interface ISoftDelete
{
    bool IsDeleted { get; set; }
}

// 필터 무시 (필요 시)
var allProducts = await context.Products
    .IgnoreQueryFilters()
    .ToListAsync();
```

### 자동 속성 구성

```csharp
// ModelBuilder 확장 메서드
public static class ModelBuilderExtensions
{
    public static void ApplyGlobalConfigurations(this ModelBuilder modelBuilder)
    {
        foreach (var entityType in modelBuilder.Model.GetEntityTypes())
        {
            // DateTime은 항상 datetime2
            foreach (var property in entityType.GetProperties()
                .Where(p => p.ClrType == typeof(DateTime) || p.ClrType == typeof(DateTime?)))
            {
                property.SetColumnType("datetime2");
            }

            // decimal 정밀도
            foreach (var property in entityType.GetProperties()
                .Where(p => p.ClrType == typeof(decimal) || p.ClrType == typeof(decimal?)))
            {
                if (property.GetPrecision() == null)
                {
                    property.SetPrecision(18);
                    property.SetScale(2);
                }
            }

            // 외래 키 인덱스
            foreach (var foreignKey in entityType.GetForeignKeys())
            {
                var index = entityType.FindIndex(foreignKey.Properties);
                if (index == null)
                {
                    entityType.AddIndex(foreignKey.Properties);
                }
            }
        }
    }
}

// 사용
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);
    modelBuilder.ApplyGlobalConfigurations();
}
```

---

## 요약

| 항목 | 핵심 내용 |
|------|----------|
| OnModelCreating | 모델 구성의 진입점, 구조화 권장 |
| IEntityTypeConfiguration | 엔티티별 구성 분리, 재사용성 |
| 인덱스 | 단일/복합, 유니크, 필터링, 포함 열 |
| 복합 키 | 두 개 이상 속성으로 구성, Fluent API 필수 |
| 글로벌 설정 | 모든 엔티티에 일괄 적용 |

## 다음 장 예고

다음 장에서는 상속 매핑 전략(TPH, TPT, TPC)과 각 전략의 장단점을 알아봅니다.
