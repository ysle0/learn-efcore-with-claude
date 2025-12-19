# Chapter 25-28: 고급 주제

## 개요

데이터베이스 프로바이더별 기능, JSON 컬럼, 시간 테이블, Database-First 접근법을 알아봅니다.

---

## 25. 데이터베이스 프로바이더별 기능

### SQL Server 전용

```csharp
// Temporal Tables
modelBuilder.Entity<Product>(entity =>
{
    entity.ToTable("Products", t => t.IsTemporal());
});

// 히스토리 쿼리
var history = await context.Products
    .TemporalAll()
    .Where(p => p.Id == 1)
    .ToListAsync();

var pointInTime = await context.Products
    .TemporalAsOf(new DateTime(2024, 1, 1))
    .ToListAsync();

// HierarchyId
public class Category
{
    public HierarchyId Path { get; set; }
}

// 전체 텍스트 검색
.Where(p => EF.Functions.FreeText(p.Description, "wireless"))
```

### PostgreSQL 전용 (Npgsql)

```csharp
// 배열 타입
public class Post
{
    public string[] Tags { get; set; } = Array.Empty<string>();
}

modelBuilder.Entity<Post>()
    .Property(p => p.Tags)
    .HasColumnType("text[]");

// 배열 쿼리
.Where(p => p.Tags.Contains("efcore"))

// JSON 타입
.Property(p => p.Metadata)
    .HasColumnType("jsonb");

// 범위 타입
public NpgsqlRange<DateTime> ValidRange { get; set; }

// 확장 기능
modelBuilder.HasPostgresExtension("uuid-ossp");
modelBuilder.HasPostgresExtension("hstore");
```

---

## 26. JSON 컬럼 지원 (EF Core 7+)

### 기본 사용

```csharp
public class Order
{
    public int Id { get; set; }
    public ShippingInfo Shipping { get; set; } = null!;
}

public class ShippingInfo
{
    public string Carrier { get; set; } = string.Empty;
    public Address Address { get; set; } = null!;
}

// 매핑
modelBuilder.Entity<Order>()
    .OwnsOne(o => o.Shipping, shipping =>
    {
        shipping.ToJson();
        shipping.OwnsOne(s => s.Address);
    });
```

### JSON 쿼리

```csharp
// JSON 속성으로 필터링
var orders = await context.Orders
    .Where(o => o.Shipping.Carrier == "FedEx")
    .ToListAsync();

var seoulOrders = await context.Orders
    .Where(o => o.Shipping.Address.City == "Seoul")
    .ToListAsync();

// JSON 컬렉션
modelBuilder.Entity<Product>()
    .OwnsMany(p => p.Attributes, attr =>
    {
        attr.ToJson();
    });

// 컬렉션 쿼리
.Where(p => p.Attributes.Any(a => a.Name == "Color"))
```

---

## 27. 시간 테이블과 감사

### 자동 감사 로그

```csharp
public interface IAuditable
{
    DateTime CreatedAt { get; set; }
    DateTime? ModifiedAt { get; set; }
    string CreatedBy { get; set; }
    string? ModifiedBy { get; set; }
}

public override async Task<int> SaveChangesAsync(CancellationToken ct = default)
{
    var now = DateTime.UtcNow;
    var userId = _currentUser.Id;

    foreach (var entry in ChangeTracker.Entries<IAuditable>())
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

    return await base.SaveChangesAsync(ct);
}
```

### Shadow Property 타임스탬프

```csharp
modelBuilder.Entity<Product>()
    .Property<DateTime>("LastModified")
    .HasDefaultValueSql("GETUTCDATE()");

// 업데이트 시 자동 갱신
foreach (var entry in ChangeTracker.Entries()
    .Where(e => e.State == EntityState.Modified))
{
    entry.Property("LastModified").CurrentValue = DateTime.UtcNow;
}
```

---

## 28. Database-First 접근법

### Scaffold 명령

```bash
# 전체 데이터베이스
dotnet ef dbcontext scaffold \
    "Server=localhost;Database=MyDb;..." \
    Microsoft.EntityFrameworkCore.SqlServer \
    --output-dir Models \
    --context-dir Data \
    --context MyDbContext

# 특정 테이블만
dotnet ef dbcontext scaffold \
    "connection-string" \
    Microsoft.EntityFrameworkCore.SqlServer \
    --table Products --table Categories

# 옵션
--force                    # 기존 파일 덮어쓰기
--no-onconfiguring         # OnConfiguring 생성 안 함
--use-database-names       # DB 이름 그대로 사용
--data-annotations         # Data Annotations 사용
```

### 생성된 코드 커스터마이징

```csharp
// 생성된 파일: Models/Product.cs
public partial class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = null!;
}

// 확장 파일: Models/Product.Extended.cs
public partial class Product
{
    // 추가 로직
    public string DisplayName => $"{Name} (ID: {Id})";

    public void Validate()
    {
        if (string.IsNullOrEmpty(Name))
            throw new ValidationException("Name is required");
    }
}
```

### 스키마 변경 동기화

```bash
# 스키마 변경 후 재생성
dotnet ef dbcontext scaffold ... --force

# 또는 마이그레이션 전환
dotnet ef migrations add InitialCreate
# 기존 테이블이 있으면 빈 마이그레이션 생성
```

---

## 요약

| 기능 | 프로바이더 | 사용 시나리오 |
|------|-----------|-------------|
| Temporal Tables | SQL Server | 이력 추적 |
| 배열 타입 | PostgreSQL | 태그, 목록 |
| JSON 컬럼 | 모든 주요 DB | 유연한 스키마 |
| Database-First | 모든 DB | 레거시 시스템 |
