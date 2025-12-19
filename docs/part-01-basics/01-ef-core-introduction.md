# Chapter 01: EF Core 소개

## 개요

Entity Framework Core(EF Core)는 .NET 애플리케이션을 위한 현대적인 객체-관계 매핑(ORM) 프레임워크입니다. 이 장에서는 ORM의 기본 개념부터 EF Core의 특징과 장단점까지 알아봅니다.

---

## 1.1 ORM(Object-Relational Mapping)이란?

### 간략 설명
ORM은 객체 지향 프로그래밍 언어의 객체와 관계형 데이터베이스의 테이블 간의 데이터를 자동으로 매핑해주는 기술입니다.

### 상세 설명

#### ORM이 해결하는 문제: 임피던스 불일치(Impedance Mismatch)

객체 지향 프로그래밍과 관계형 데이터베이스는 근본적으로 다른 패러다임을 가지고 있습니다:

| 구분 | 객체 지향 | 관계형 데이터베이스 |
|------|----------|-------------------|
| 데이터 구조 | 객체(클래스) | 테이블(행과 열) |
| 관계 표현 | 참조(Reference) | 외래 키(Foreign Key) |
| 상속 | 지원 | 직접 지원하지 않음 |
| 식별 | 객체 동일성 | 기본 키(Primary Key) |
| 캡슐화 | 메서드 포함 | 데이터만 저장 |

#### ORM 없이 데이터 접근하기 (ADO.NET)

```csharp
// 전통적인 ADO.NET 방식
using var connection = new SqlConnection(connectionString);
await connection.OpenAsync();

using var command = new SqlCommand(
    "SELECT Id, Name, Email FROM Users WHERE Id = @Id",
    connection);
command.Parameters.AddWithValue("@Id", userId);

using var reader = await command.ExecuteReaderAsync();
if (await reader.ReadAsync())
{
    var user = new User
    {
        Id = reader.GetInt32(0),
        Name = reader.GetString(1),
        Email = reader.GetString(2)
    };
}
```

**문제점:**
- 반복적인 보일러플레이트 코드
- SQL 문자열과 C# 코드 분리
- 타입 안전성 부족
- 유지보수 어려움

#### ORM을 사용한 데이터 접근 (EF Core)

```csharp
// EF Core 방식
var user = await context.Users.FindAsync(userId);
```

**장점:**
- 간결하고 직관적인 코드
- 컴파일 타임 타입 검사
- SQL 자동 생성
- 생산성 향상

#### ORM의 동작 원리

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   C# Objects    │ ←→  │      ORM        │ ←→  │    Database     │
│                 │     │   (EF Core)     │     │                 │
│  - User         │     │  - 쿼리 변환     │     │  - Users 테이블  │
│  - Order        │     │  - 변경 추적     │     │  - Orders 테이블 │
│  - Product      │     │  - 매핑 관리     │     │  - Products     │
└─────────────────┘     └─────────────────┘     └─────────────────┘
```

---

## 1.2 EF Core vs EF6 비교

### 간략 설명
EF Core는 Entity Framework 6(EF6)의 후속 버전으로, 처음부터 새로 설계되었습니다. 더 가볍고, 확장 가능하며, 크로스 플랫폼을 지원합니다.

### 상세 비교

#### 아키텍처 차이

| 특성 | EF6 | EF Core |
|------|-----|---------|
| 플랫폼 | .NET Framework only | 크로스 플랫폼 (.NET Core, .NET 5+) |
| 설계 | 모놀리식 | 모듈식, 경량화 |
| 성능 | 상대적으로 무거움 | 최적화됨 |
| 오픈소스 | 부분적 | 완전한 오픈소스 |
| 활발한 개발 | 유지보수 모드 | 적극적 개발 |

#### EF6에서 제거된 기능

```csharp
// ❌ EF Core에서 지원하지 않는 EF6 기능들

// 1. EDMX 디자이너 (시각적 모델 디자인)
// EF Core는 Code-First만 지원

// 2. ObjectContext API
// EF Core는 DbContext만 사용

// 3. Entity SQL
// var query = "SELECT VALUE c FROM Container.Customers AS c";
// EF Core는 LINQ와 Raw SQL만 지원

// 4. 자동 마이그레이션
// Database.SetInitializer(new MigrateDatabaseToLatestVersion<...>());
// EF Core는 명시적 마이그레이션 사용
```

#### EF Core에서 새로 추가된 기능

```csharp
// ✅ EF Core 전용 기능들

// 1. 배치 작업 (Batch Operations) - EF Core 7+
await context.Users
    .Where(u => u.IsInactive)
    .ExecuteDeleteAsync();

await context.Products
    .Where(p => p.Category == "Electronics")
    .ExecuteUpdateAsync(s => s.SetProperty(p => p.Discount, 0.1m));

// 2. 전역 쿼리 필터 (Global Query Filters)
modelBuilder.Entity<Post>()
    .HasQueryFilter(p => !p.IsDeleted);

// 3. 소유 타입 (Owned Types)
modelBuilder.Entity<Order>()
    .OwnsOne(o => o.ShippingAddress);

// 4. 테이블 분할 (Table Splitting)
modelBuilder.Entity<Customer>()
    .ToTable("Customers");
modelBuilder.Entity<CustomerDetails>()
    .ToTable("Customers");

// 5. 값 변환기 (Value Converters)
modelBuilder.Entity<Rider>()
    .Property(r => r.Mount)
    .HasConversion<string>();

// 6. DbContext 풀링
services.AddDbContextPool<BloggingContext>(
    options => options.UseSqlServer(connectionString));
```

#### 버전별 주요 기능

| 버전 | 주요 기능 |
|------|----------|
| EF Core 2.0 | 테이블 분할, 소유 타입, 전역 쿼리 필터 |
| EF Core 3.0 | 단일 쿼리 모드, Cosmos DB 지원 |
| EF Core 5.0 | 다대다 관계 간소화, 분할 쿼리 |
| EF Core 6.0 | Temporal Tables, 마이그레이션 번들 |
| EF Core 7.0 | ExecuteUpdate/Delete, JSON 컬럼 |
| EF Core 8.0 | Complex Types, 원시 SQL 개선 |
| EF Core 9.0 | LINQ 개선, 성능 최적화 |

#### 마이그레이션 가이드

EF6에서 EF Core로 마이그레이션할 때 고려사항:

```csharp
// EF6 코드
public class BloggingContext : DbContext
{
    public BloggingContext() : base("name=BloggingDatabase")
    {
    }

    public DbSet<Blog> Blogs { get; set; }
}

// EF Core로 변환
public class BloggingContext : DbContext
{
    public BloggingContext(DbContextOptions<BloggingContext> options)
        : base(options)
    {
    }

    public DbSet<Blog> Blogs { get; set; }
}

// Program.cs 또는 Startup.cs에서 설정
services.AddDbContext<BloggingContext>(options =>
    options.UseSqlServer(Configuration.GetConnectionString("BloggingDatabase")));
```

---

## 1.3 EF Core의 장점과 단점

### 간략 설명
EF Core는 강력한 기능을 제공하지만, 모든 상황에 적합한 것은 아닙니다. 장단점을 이해하고 적절한 상황에 사용해야 합니다.

### 장점 상세

#### 1. 생산성 향상

```csharp
// 복잡한 조인 쿼리도 간단하게
var ordersWithDetails = await context.Orders
    .Include(o => o.Customer)
    .Include(o => o.OrderItems)
        .ThenInclude(oi => oi.Product)
    .Where(o => o.OrderDate >= startDate)
    .OrderByDescending(o => o.TotalAmount)
    .Take(10)
    .ToListAsync();

// SQL로 직접 작성하면 수십 줄이 될 코드를 몇 줄로 해결
```

#### 2. 타입 안전성

```csharp
// 컴파일 타임에 오류 검출
var users = context.Users
    .Where(u => u.Naem == "John")  // ❌ 컴파일 에러: 'Naem' 오타
    .ToList();

// SQL 문자열은 런타임에만 오류 발견
// "SELECT * FROM Users WHERE Naem = 'John'"  // 런타임 에러
```

#### 3. 데이터베이스 독립성

```csharp
// 동일한 코드로 여러 데이터베이스 지원
public void ConfigureServices(IServiceCollection services)
{
    services.AddDbContext<AppDbContext>(options =>
    {
        var provider = Configuration["DatabaseProvider"];

        switch (provider)
        {
            case "SqlServer":
                options.UseSqlServer(Configuration.GetConnectionString("SqlServer"));
                break;
            case "PostgreSQL":
                options.UseNpgsql(Configuration.GetConnectionString("PostgreSQL"));
                break;
            case "SQLite":
                options.UseSqlite(Configuration.GetConnectionString("SQLite"));
                break;
        }
    });
}
```

#### 4. 변경 추적 자동화

```csharp
// 엔티티 수정 시 자동으로 변경 감지
var user = await context.Users.FindAsync(1);
user.Name = "Updated Name";  // 변경 추적됨
user.Email = "new@email.com";  // 변경 추적됨

await context.SaveChangesAsync();  // UPDATE 쿼리 자동 생성
```

#### 5. 마이그레이션 지원

```bash
# 모델 변경 시 마이그레이션 자동 생성
dotnet ef migrations add AddUserPhoneNumber

# 데이터베이스에 적용
dotnet ef database update
```

### 단점 상세

#### 1. 학습 곡선

```csharp
// 초보자가 흔히 저지르는 실수들

// ❌ N+1 쿼리 문제
var blogs = context.Blogs.ToList();
foreach (var blog in blogs)
{
    Console.WriteLine(blog.Posts.Count);  // 각 블로그마다 추가 쿼리 발생!
}

// ✅ 올바른 방법
var blogs = context.Blogs
    .Include(b => b.Posts)
    .ToList();
```

#### 2. 성능 오버헤드

```csharp
// 간단한 대량 삽입에서 EF Core vs Raw SQL 성능 차이

// EF Core (느림 - 각 레코드마다 INSERT)
foreach (var item in items)
{
    context.Items.Add(item);
}
await context.SaveChangesAsync();

// Raw SQL 또는 SqlBulkCopy (빠름)
using var bulkCopy = new SqlBulkCopy(connection);
bulkCopy.DestinationTableName = "Items";
await bulkCopy.WriteToServerAsync(dataTable);

// EF Core 7+ 해결책
await context.BulkInsertAsync(items);  // 서드파티 라이브러리
```

#### 3. 복잡한 쿼리의 한계

```csharp
// EF Core로 표현하기 어려운 복잡한 SQL

// 재귀 CTE (Common Table Expression)
// EF Core는 직접 지원하지 않음
var sql = @"
    WITH RECURSIVE CategoryTree AS (
        SELECT Id, Name, ParentId, 0 as Level
        FROM Categories
        WHERE ParentId IS NULL
        UNION ALL
        SELECT c.Id, c.Name, c.ParentId, ct.Level + 1
        FROM Categories c
        JOIN CategoryTree ct ON c.ParentId = ct.Id
    )
    SELECT * FROM CategoryTree";

var categories = await context.Categories
    .FromSqlRaw(sql)
    .ToListAsync();
```

#### 4. 추상화 누수

```csharp
// 데이터베이스별 동작 차이

// SQL Server: 대소문자 구분 없음 (기본 collation)
// PostgreSQL: 대소문자 구분함

var users = context.Users
    .Where(u => u.Name == "john")  // SQL Server: John, JOHN 등 모두 매칭
    .ToList();                      // PostgreSQL: 정확히 "john"만 매칭

// 해결책: 명시적으로 처리
var users = context.Users
    .Where(u => u.Name.ToLower() == "john".ToLower())
    .ToList();

// 또는 EF.Functions 사용
var users = context.Users
    .Where(u => EF.Functions.ILike(u.Name, "john"))  // PostgreSQL
    .ToList();
```

### 사용 권장 상황

| 상황 | EF Core 권장 | 대안 |
|------|-------------|------|
| 일반적인 CRUD 애플리케이션 | ✅ 강력 추천 | - |
| 도메인 중심 설계 (DDD) | ✅ 추천 | - |
| 복잡한 보고서/분석 | ⚠️ 부분 사용 | Raw SQL, Dapper |
| 대용량 배치 처리 | ⚠️ 주의 필요 | SqlBulkCopy, Dapper |
| 마이크로초 단위 지연시간 필요 | ❌ 비추천 | Dapper, ADO.NET |
| 레거시 DB 스키마 | ⚠️ 상황에 따라 | Database-First 접근 |

---

## 1.4 지원되는 데이터베이스 프로바이더

### 간략 설명
EF Core는 플러그인 방식의 데이터베이스 프로바이더를 통해 다양한 데이터베이스를 지원합니다.

### 공식 프로바이더

#### Microsoft 공식 지원

| 프로바이더 | NuGet 패키지 | 용도 |
|-----------|-------------|------|
| SQL Server | `Microsoft.EntityFrameworkCore.SqlServer` | 프로덕션 |
| SQLite | `Microsoft.EntityFrameworkCore.Sqlite` | 개발/테스트, 모바일 |
| In-Memory | `Microsoft.EntityFrameworkCore.InMemory` | 단위 테스트 |
| Cosmos DB | `Microsoft.EntityFrameworkCore.Cosmos` | NoSQL |

```csharp
// SQL Server 설정
services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(
        connectionString,
        sqlOptions =>
        {
            sqlOptions.EnableRetryOnFailure(
                maxRetryCount: 3,
                maxRetryDelay: TimeSpan.FromSeconds(30),
                errorNumbersToAdd: null);
            sqlOptions.CommandTimeout(30);
        }));

// SQLite 설정
services.AddDbContext<AppDbContext>(options =>
    options.UseSqlite("Data Source=app.db"));

// In-Memory 설정 (테스트용)
services.AddDbContext<AppDbContext>(options =>
    options.UseInMemoryDatabase("TestDatabase"));

// Cosmos DB 설정
services.AddDbContext<AppDbContext>(options =>
    options.UseCosmos(
        accountEndpoint: "https://localhost:8081",
        accountKey: "your-key",
        databaseName: "FamilyDatabase"));
```

### 서드파티 프로바이더

#### 주요 프로바이더

| 데이터베이스 | NuGet 패키지 | 비고 |
|-------------|-------------|------|
| PostgreSQL | `Npgsql.EntityFrameworkCore.PostgreSQL` | 가장 활발한 커뮤니티 |
| MySQL | `Pomelo.EntityFrameworkCore.MySql` | 권장 MySQL 프로바이더 |
| MySQL | `MySql.EntityFrameworkCore` | Oracle 공식 |
| Oracle | `Oracle.EntityFrameworkCore` | Oracle 공식 |
| MariaDB | `Pomelo.EntityFrameworkCore.MySql` | MySQL과 동일 |

```csharp
// PostgreSQL 설정
services.AddDbContext<AppDbContext>(options =>
    options.UseNpgsql(
        connectionString,
        npgsqlOptions =>
        {
            npgsqlOptions.EnableRetryOnFailure();
            npgsqlOptions.CommandTimeout(30);
        }));

// MySQL (Pomelo) 설정
services.AddDbContext<AppDbContext>(options =>
    options.UseMySql(
        connectionString,
        ServerVersion.AutoDetect(connectionString),
        mySqlOptions =>
        {
            mySqlOptions.EnableRetryOnFailure();
        }));

// Oracle 설정
services.AddDbContext<AppDbContext>(options =>
    options.UseOracle(connectionString));
```

### 프로바이더별 특수 기능

```csharp
// SQL Server 전용 기능
modelBuilder.Entity<Blog>()
    .Property(b => b.Url)
    .IsUnicode(false);  // VARCHAR 사용

modelBuilder.Entity<Person>()
    .ToTable("People", t => t.IsTemporal());  // Temporal Table

// PostgreSQL 전용 기능
modelBuilder.Entity<Blog>()
    .Property(b => b.Tags)
    .HasColumnType("text[]");  // 배열 타입

modelBuilder.HasPostgresExtension("hstore");  // 확장 기능

// MySQL 전용 기능
modelBuilder.Entity<Blog>()
    .Property(b => b.Id)
    .UseMySqlIdentityColumn();  // AUTO_INCREMENT
```

### 프로바이더 선택 가이드

```
프로바이더 선택 흐름도:

시작
  │
  ├─ 프로덕션 환경?
  │   ├─ Yes → 엔터프라이즈 환경?
  │   │         ├─ Yes → SQL Server 또는 Oracle
  │   │         └─ No  → PostgreSQL (오픈소스, 고성능)
  │   │                   또는 MySQL (널리 사용됨)
  │   │
  │   └─ No → 개발/테스트용?
  │           ├─ 로컬 개발 → SQLite
  │           └─ 단위 테스트 → In-Memory
  │                           (주의: 관계형 DB 기능 제한)
  │
  └─ 클라우드 네이티브?
      ├─ Azure → SQL Server, Cosmos DB
      ├─ AWS → PostgreSQL (RDS), MySQL (RDS)
      └─ GCP → PostgreSQL (Cloud SQL)
```

---

## 요약

| 항목 | 핵심 내용 |
|------|----------|
| ORM | 객체와 데이터베이스 간의 자동 매핑 기술 |
| EF Core 특징 | 경량, 크로스플랫폼, 모듈식 설계 |
| EF6 대비 장점 | 성능 향상, 새로운 기능, 활발한 개발 |
| 장점 | 생산성, 타입 안전성, DB 독립성 |
| 단점 | 학습 곡선, 복잡한 쿼리 한계, 성능 오버헤드 |
| 프로바이더 | SQL Server, PostgreSQL, MySQL, SQLite 등 |

## 다음 장 예고

다음 장에서는 EF Core 개발 환경을 설정하고 첫 번째 프로젝트를 생성하는 방법을 알아봅니다.
