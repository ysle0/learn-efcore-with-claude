# Chapter 02: 개발 환경 설정

## 개요

EF Core 개발을 시작하기 위한 환경 설정 방법을 단계별로 알아봅니다. .NET SDK 설치부터 첫 번째 프로젝트 생성, DbContext 구성까지 실습합니다.

---

## 2.1 .NET SDK 설치

### 간략 설명
EF Core는 .NET 플랫폼에서 동작하므로 .NET SDK가 필요합니다. 최신 LTS(Long Term Support) 버전 설치를 권장합니다.

### 상세 설명

#### 지원 버전 매트릭스

| EF Core 버전 | 최소 .NET 버전 | 권장 버전 | 지원 종료 |
|-------------|---------------|----------|----------|
| EF Core 9.0 | .NET 8 | .NET 9 | 2026년 5월 |
| EF Core 8.0 | .NET 8 | .NET 8 | 2026년 11월 |
| EF Core 7.0 | .NET 6 | .NET 7 | 2024년 5월 (종료) |
| EF Core 6.0 | .NET 6 | .NET 6 | 2024년 11월 (종료) |

#### Windows 설치

```powershell
# 1. winget을 사용한 설치 (Windows 10 이상)
winget install Microsoft.DotNet.SDK.8

# 2. 또는 Chocolatey 사용
choco install dotnet-sdk

# 3. 설치 확인
dotnet --version
dotnet --list-sdks
```

#### macOS 설치

```bash
# 1. Homebrew 사용
brew install --cask dotnet-sdk

# 2. 또는 공식 설치 스크립트
curl -sSL https://dot.net/v1/dotnet-install.sh | bash /dev/stdin --channel LTS

# 3. PATH 설정 (~/.zshrc 또는 ~/.bashrc)
export DOTNET_ROOT=$HOME/.dotnet
export PATH=$PATH:$DOTNET_ROOT:$DOTNET_ROOT/tools

# 4. 설치 확인
dotnet --version
```

#### Linux 설치 (Ubuntu/Debian)

```bash
# 1. Microsoft 패키지 저장소 추가
wget https://packages.microsoft.com/config/ubuntu/22.04/packages-microsoft-prod.deb -O packages-microsoft-prod.deb
sudo dpkg -i packages-microsoft-prod.deb
rm packages-microsoft-prod.deb

# 2. SDK 설치
sudo apt-get update
sudo apt-get install -y dotnet-sdk-8.0

# 3. 설치 확인
dotnet --version
```

#### Docker 환경

```dockerfile
# 개발용 Dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS development

WORKDIR /app

# EF Core 도구 설치
RUN dotnet tool install --global dotnet-ef

# PATH에 도구 추가
ENV PATH="$PATH:/root/.dotnet/tools"

# 소스 코드 복사 및 복원
COPY *.csproj ./
RUN dotnet restore

COPY . ./
```

---

## 2.2 필요한 NuGet 패키지

### 간략 설명
EF Core는 모듈식으로 설계되어 필요한 패키지만 선택적으로 설치할 수 있습니다.

### 핵심 패키지

```xml
<!-- 프로젝트 파일 (.csproj) -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
  </PropertyGroup>

  <ItemGroup>
    <!-- EF Core 핵심 패키지 (프로바이더에 포함됨) -->
    <!-- <PackageReference Include="Microsoft.EntityFrameworkCore" Version="8.0.0" /> -->

    <!-- 데이터베이스 프로바이더 (하나 선택) -->
    <PackageReference Include="Microsoft.EntityFrameworkCore.SqlServer" Version="8.0.0" />
    <!-- 또는 -->
    <!-- <PackageReference Include="Npgsql.EntityFrameworkCore.PostgreSQL" Version="8.0.0" /> -->
    <!-- <PackageReference Include="Microsoft.EntityFrameworkCore.Sqlite" Version="8.0.0" /> -->

    <!-- 디자인 타임 도구 (마이그레이션 생성에 필요) -->
    <PackageReference Include="Microsoft.EntityFrameworkCore.Design" Version="8.0.0">
      <PrivateAssets>all</PrivateAssets>
      <IncludeAssets>runtime; build; native; contentfiles; analyzers</IncludeAssets>
    </PackageReference>
  </ItemGroup>
</Project>
```

### 패키지 설치 명령어

```bash
# .NET CLI 사용

# SQL Server 프로바이더
dotnet add package Microsoft.EntityFrameworkCore.SqlServer

# PostgreSQL 프로바이더
dotnet add package Npgsql.EntityFrameworkCore.PostgreSQL

# SQLite 프로바이더
dotnet add package Microsoft.EntityFrameworkCore.Sqlite

# 디자인 타임 도구 (마이그레이션용)
dotnet add package Microsoft.EntityFrameworkCore.Design

# EF Core CLI 도구 (전역 설치)
dotnet tool install --global dotnet-ef

# 도구 업데이트
dotnet tool update --global dotnet-ef
```

### 패키지 구조 이해

```
┌─────────────────────────────────────────────────────────────┐
│                    애플리케이션 코드                          │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│           Microsoft.EntityFrameworkCore.SqlServer           │
│  ┌────────────────────────────────────────────────────────┐ │
│  │         Microsoft.EntityFrameworkCore.Relational       │ │
│  │  ┌──────────────────────────────────────────────────┐  │ │
│  │  │           Microsoft.EntityFrameworkCore          │  │ │
│  │  │  ┌────────────────────────────────────────────┐  │  │ │
│  │  │  │    Microsoft.EntityFrameworkCore.Abstractions │  │ │
│  │  │  └────────────────────────────────────────────┘  │  │ │
│  │  └──────────────────────────────────────────────────┘  │ │
│  └────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

### 선택적 패키지

```xml
<ItemGroup>
  <!-- 프록시 기반 지연 로딩 -->
  <PackageReference Include="Microsoft.EntityFrameworkCore.Proxies" Version="8.0.0" />

  <!-- In-Memory 데이터베이스 (테스트용) -->
  <PackageReference Include="Microsoft.EntityFrameworkCore.InMemory" Version="8.0.0" />

  <!-- 분석기 (코드 품질) -->
  <PackageReference Include="Microsoft.EntityFrameworkCore.Analyzers" Version="8.0.0" />
</ItemGroup>
```

---

## 2.3 첫 번째 EF Core 프로젝트 생성

### 간략 설명
콘솔 애플리케이션을 생성하고 EF Core를 설정하여 기본적인 CRUD 작업을 수행합니다.

### 프로젝트 생성

```bash
# 1. 솔루션 폴더 생성
mkdir EFCoreDemo
cd EFCoreDemo

# 2. 솔루션 파일 생성
dotnet new sln -n EFCoreDemo

# 3. 콘솔 프로젝트 생성
dotnet new console -n EFCoreDemo.ConsoleApp
dotnet sln add EFCoreDemo.ConsoleApp

# 4. 클래스 라이브러리 생성 (데이터 계층)
dotnet new classlib -n EFCoreDemo.Data
dotnet sln add EFCoreDemo.Data

# 5. 프로젝트 참조 추가
cd EFCoreDemo.ConsoleApp
dotnet add reference ../EFCoreDemo.Data

# 6. 패키지 설치
cd ../EFCoreDemo.Data
dotnet add package Microsoft.EntityFrameworkCore.SqlServer
dotnet add package Microsoft.EntityFrameworkCore.Design
```

### 프로젝트 구조

```
EFCoreDemo/
├── EFCoreDemo.sln
├── EFCoreDemo.ConsoleApp/
│   ├── EFCoreDemo.ConsoleApp.csproj
│   └── Program.cs
└── EFCoreDemo.Data/
    ├── EFCoreDemo.Data.csproj
    ├── Entities/
    │   ├── Blog.cs
    │   └── Post.cs
    └── BloggingContext.cs
```

### 엔티티 클래스 작성

```csharp
// EFCoreDemo.Data/Entities/Blog.cs
namespace EFCoreDemo.Data.Entities;

public class Blog
{
    public int Id { get; set; }
    public string Url { get; set; } = string.Empty;
    public string Title { get; set; } = string.Empty;
    public DateTime CreatedAt { get; set; }

    // 네비게이션 속성
    public ICollection<Post> Posts { get; set; } = new List<Post>();
}
```

```csharp
// EFCoreDemo.Data/Entities/Post.cs
namespace EFCoreDemo.Data.Entities;

public class Post
{
    public int Id { get; set; }
    public string Title { get; set; } = string.Empty;
    public string Content { get; set; } = string.Empty;
    public DateTime PublishedAt { get; set; }

    // 외래 키
    public int BlogId { get; set; }

    // 네비게이션 속성
    public Blog Blog { get; set; } = null!;
}
```

### DbContext 작성

```csharp
// EFCoreDemo.Data/BloggingContext.cs
using EFCoreDemo.Data.Entities;
using Microsoft.EntityFrameworkCore;

namespace EFCoreDemo.Data;

public class BloggingContext : DbContext
{
    // DbSet 속성 - 테이블과 매핑
    public DbSet<Blog> Blogs => Set<Blog>();
    public DbSet<Post> Posts => Set<Post>();

    // 연결 문자열 설정
    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        optionsBuilder.UseSqlServer(
            "Server=localhost;Database=BloggingDb;Trusted_Connection=True;TrustServerCertificate=True");
    }

    // 모델 구성
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Blog 엔티티 구성
        modelBuilder.Entity<Blog>(entity =>
        {
            entity.HasKey(b => b.Id);
            entity.Property(b => b.Url).HasMaxLength(500).IsRequired();
            entity.Property(b => b.Title).HasMaxLength(200).IsRequired();
            entity.Property(b => b.CreatedAt).HasDefaultValueSql("GETUTCDATE()");
        });

        // Post 엔티티 구성
        modelBuilder.Entity<Post>(entity =>
        {
            entity.HasKey(p => p.Id);
            entity.Property(p => p.Title).HasMaxLength(200).IsRequired();

            // 관계 설정
            entity.HasOne(p => p.Blog)
                  .WithMany(b => b.Posts)
                  .HasForeignKey(p => p.BlogId)
                  .OnDelete(DeleteBehavior.Cascade);
        });
    }
}
```

### Program.cs 작성

```csharp
// EFCoreDemo.ConsoleApp/Program.cs
using EFCoreDemo.Data;
using EFCoreDemo.Data.Entities;
using Microsoft.EntityFrameworkCore;

Console.WriteLine("=== EF Core Demo ===\n");

// DbContext 생성
using var context = new BloggingContext();

// 1. 데이터베이스 생성 (개발용)
Console.WriteLine("데이터베이스 확인 중...");
await context.Database.EnsureCreatedAsync();
Console.WriteLine("데이터베이스 준비 완료!\n");

// 2. 데이터 추가 (Create)
Console.WriteLine("--- 데이터 추가 ---");
var blog = new Blog
{
    Url = "https://devblog.com",
    Title = "개발 블로그",
    CreatedAt = DateTime.UtcNow,
    Posts = new List<Post>
    {
        new Post
        {
            Title = "EF Core 시작하기",
            Content = "EF Core는 .NET을 위한 현대적인 ORM입니다.",
            PublishedAt = DateTime.UtcNow
        },
        new Post
        {
            Title = "LINQ 마스터하기",
            Content = "LINQ를 활용한 데이터 쿼리 방법을 알아봅니다.",
            PublishedAt = DateTime.UtcNow
        }
    }
};

context.Blogs.Add(blog);
await context.SaveChangesAsync();
Console.WriteLine($"블로그 추가됨: {blog.Title} (ID: {blog.Id})\n");

// 3. 데이터 조회 (Read)
Console.WriteLine("--- 데이터 조회 ---");
var blogs = await context.Blogs
    .Include(b => b.Posts)
    .ToListAsync();

foreach (var b in blogs)
{
    Console.WriteLine($"블로그: {b.Title} ({b.Url})");
    foreach (var post in b.Posts)
    {
        Console.WriteLine($"  - 포스트: {post.Title}");
    }
}
Console.WriteLine();

// 4. 데이터 수정 (Update)
Console.WriteLine("--- 데이터 수정 ---");
var blogToUpdate = await context.Blogs.FirstAsync();
blogToUpdate.Title = "업데이트된 개발 블로그";
await context.SaveChangesAsync();
Console.WriteLine($"블로그 제목 수정됨: {blogToUpdate.Title}\n");

// 5. 데이터 삭제 (Delete)
Console.WriteLine("--- 데이터 삭제 ---");
var postToDelete = await context.Posts.FirstAsync();
context.Posts.Remove(postToDelete);
await context.SaveChangesAsync();
Console.WriteLine($"포스트 삭제됨: {postToDelete.Title}\n");

Console.WriteLine("=== 완료 ===");
```

### 마이그레이션 생성 및 적용

```bash
# 프로젝트 디렉토리에서 실행
cd EFCoreDemo.Data

# 첫 번째 마이그레이션 생성
dotnet ef migrations add InitialCreate --startup-project ../EFCoreDemo.ConsoleApp

# 마이그레이션 적용
dotnet ef database update --startup-project ../EFCoreDemo.ConsoleApp

# 애플리케이션 실행
cd ../EFCoreDemo.ConsoleApp
dotnet run
```

---

## 2.4 DbContext 기본 구성

### 간략 설명
DbContext는 EF Core의 핵심 클래스로, 데이터베이스와의 모든 상호작용을 관리합니다.

### DbContext의 역할

```
┌──────────────────────────────────────────────────────────────┐
│                        DbContext                              │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  변경 추적 (Change Tracking)                            │  │
│  │  - 엔티티 상태 관리                                      │  │
│  │  - 변경 감지                                            │  │
│  └────────────────────────────────────────────────────────┘  │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  쿼리 (Querying)                                        │  │
│  │  - LINQ to SQL 변환                                     │  │
│  │  - 결과 매핑                                            │  │
│  └────────────────────────────────────────────────────────┘  │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  저장 (Saving)                                          │  │
│  │  - INSERT/UPDATE/DELETE 생성                            │  │
│  │  - 트랜잭션 관리                                         │  │
│  └────────────────────────────────────────────────────────┘  │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  캐싱 (First-Level Cache)                               │  │
│  │  - 동일 요청 내 엔티티 캐싱                               │  │
│  └────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

### 구성 방법 1: OnConfiguring 오버라이드

```csharp
public class BloggingContext : DbContext
{
    public DbSet<Blog> Blogs => Set<Blog>();

    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        // 간단한 설정
        optionsBuilder.UseSqlServer("Server=...;Database=...;");

        // 상세 설정
        optionsBuilder
            .UseSqlServer(
                "Server=localhost;Database=BloggingDb;Trusted_Connection=True;",
                options =>
                {
                    options.EnableRetryOnFailure(3);
                    options.CommandTimeout(30);
                    options.MigrationsHistoryTable("__EFMigrationsHistory", "dbo");
                })
            .EnableSensitiveDataLogging()  // 개발 환경에서만!
            .EnableDetailedErrors()         // 개발 환경에서만!
            .LogTo(Console.WriteLine, LogLevel.Information);
    }
}
```

### 구성 방법 2: 의존성 주입 (권장)

```csharp
// DbContext 클래스
public class BloggingContext : DbContext
{
    public BloggingContext(DbContextOptions<BloggingContext> options)
        : base(options)
    {
    }

    public DbSet<Blog> Blogs => Set<Blog>();
    public DbSet<Post> Posts => Set<Post>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // 모델 구성만 여기서
    }
}

// Program.cs 또는 Startup.cs
var builder = WebApplication.CreateBuilder(args);

// 기본 등록
builder.Services.AddDbContext<BloggingContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));

// 고급 등록
builder.Services.AddDbContext<BloggingContext>((serviceProvider, options) =>
{
    var configuration = serviceProvider.GetRequiredService<IConfiguration>();
    var environment = serviceProvider.GetRequiredService<IHostEnvironment>();

    options.UseSqlServer(
        configuration.GetConnectionString("DefaultConnection"),
        sqlOptions =>
        {
            sqlOptions.EnableRetryOnFailure(
                maxRetryCount: 3,
                maxRetryDelay: TimeSpan.FromSeconds(10),
                errorNumbersToAdd: null);
        });

    if (environment.IsDevelopment())
    {
        options.EnableSensitiveDataLogging();
        options.EnableDetailedErrors();
    }
});
```

### appsettings.json 연결 문자열

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost;Database=BloggingDb;Trusted_Connection=True;TrustServerCertificate=True",
    "PostgreSQL": "Host=localhost;Database=BloggingDb;Username=postgres;Password=secret",
    "SQLite": "Data Source=blogging.db"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.EntityFrameworkCore": "Warning",
      "Microsoft.EntityFrameworkCore.Database.Command": "Information"
    }
  }
}
```

### DbContext 풀링

```csharp
// 일반 등록 (매 요청마다 새 인스턴스)
builder.Services.AddDbContext<BloggingContext>(options =>
    options.UseSqlServer(connectionString));

// 풀링 등록 (인스턴스 재사용으로 성능 향상)
builder.Services.AddDbContextPool<BloggingContext>(options =>
    options.UseSqlServer(connectionString),
    poolSize: 128);  // 기본값: 1024
```

### DbContext 수명 주기

```csharp
// ❌ 잘못된 사용: Singleton 등록
builder.Services.AddSingleton<BloggingContext>();  // 절대 하지 마세요!

// ✅ 올바른 사용: Scoped (기본값)
builder.Services.AddDbContext<BloggingContext>(options =>
    options.UseSqlServer(connectionString));

// DbContext 사용 패턴
public class BlogService
{
    private readonly BloggingContext _context;

    // 생성자 주입
    public BlogService(BloggingContext context)
    {
        _context = context;
    }

    public async Task<List<Blog>> GetBlogsAsync()
    {
        return await _context.Blogs.ToListAsync();
    }
}

// 또는 IDbContextFactory 사용 (장기 실행 작업용)
public class BackgroundJobService
{
    private readonly IDbContextFactory<BloggingContext> _contextFactory;

    public BackgroundJobService(IDbContextFactory<BloggingContext> contextFactory)
    {
        _contextFactory = contextFactory;
    }

    public async Task ProcessJobAsync()
    {
        // 명시적으로 DbContext 생성 및 해제
        await using var context = await _contextFactory.CreateDbContextAsync();
        // 작업 수행
    }
}

// IDbContextFactory 등록
builder.Services.AddDbContextFactory<BloggingContext>(options =>
    options.UseSqlServer(connectionString));
```

### 다중 DbContext 구성

```csharp
// 읽기 전용 컨텍스트
public class ReadOnlyBloggingContext : DbContext
{
    public ReadOnlyBloggingContext(DbContextOptions<ReadOnlyBloggingContext> options)
        : base(options)
    {
        ChangeTracker.QueryTrackingBehavior = QueryTrackingBehavior.NoTracking;
    }

    public DbSet<Blog> Blogs => Set<Blog>();
}

// 쓰기용 컨텍스트
public class WriteBloggingContext : DbContext
{
    public WriteBloggingContext(DbContextOptions<WriteBloggingContext> options)
        : base(options)
    {
    }

    public DbSet<Blog> Blogs => Set<Blog>();
}

// 등록
builder.Services.AddDbContext<ReadOnlyBloggingContext>(options =>
    options.UseSqlServer(configuration.GetConnectionString("ReadReplica")));

builder.Services.AddDbContext<WriteBloggingContext>(options =>
    options.UseSqlServer(configuration.GetConnectionString("Primary")));
```

---

## 개발 도구 설정

### Visual Studio 설정

```
1. NuGet 패키지 관리자 콘솔 활성화
   Tools → NuGet Package Manager → Package Manager Console

2. EF Core 명령어 사용
   PM> Add-Migration InitialCreate
   PM> Update-Database
   PM> Script-Migration

3. SQL Server Object Explorer
   View → SQL Server Object Explorer
```

### Visual Studio Code 설정

```json
// .vscode/settings.json
{
    "dotnet.defaultSolution": "EFCoreDemo.sln",
    "omnisharp.enableRoslynAnalyzers": true,
    "omnisharp.enableEditorConfigSupport": true
}

// .vscode/tasks.json
{
    "version": "2.0.0",
    "tasks": [
        {
            "label": "ef-migration-add",
            "type": "shell",
            "command": "dotnet ef migrations add ${input:migrationName}",
            "problemMatcher": []
        },
        {
            "label": "ef-database-update",
            "type": "shell",
            "command": "dotnet ef database update",
            "problemMatcher": []
        }
    ],
    "inputs": [
        {
            "id": "migrationName",
            "type": "promptString",
            "description": "마이그레이션 이름"
        }
    ]
}
```

### JetBrains Rider 설정

```
1. EF Core 플러그인 설치
   Settings → Plugins → "Entity Framework Core UI"

2. 데이터베이스 도구
   View → Tool Windows → Database

3. 마이그레이션 도구
   프로젝트 우클릭 → Entity Framework Core → Add Migration
```

---

## 요약

| 항목 | 핵심 내용 |
|------|----------|
| .NET SDK | LTS 버전 권장 (.NET 8) |
| 필수 패키지 | 프로바이더 + Design 패키지 |
| EF Core CLI | `dotnet tool install --global dotnet-ef` |
| DbContext 구성 | 의존성 주입 방식 권장 |
| 수명 주기 | Scoped (요청당 하나) |
| 풀링 | 고성능 시나리오에서 사용 |

## 다음 장 예고

다음 장에서는 엔티티 클래스를 작성하고 Data Annotations와 Fluent API를 사용하여 모델을 구성하는 방법을 알아봅니다.
