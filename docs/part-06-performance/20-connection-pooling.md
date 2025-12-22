# Chapter 20: 연결 및 풀링

## 개요

데이터베이스 연결 관리와 풀링을 통한 성능 최적화 방법을 알아봅니다. 연결 문자열 구성, 연결 풀링, DbContext 풀링을 다룹니다.

---

## 19.1 연결 문자열 구성

### SQL Server 연결 문자열

```json
// appsettings.json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost;Database=MyApp;Trusted_Connection=True;TrustServerCertificate=True;",

    "ProductionConnection": "Server=prod-server;Database=MyApp;User Id=app_user;Password=secure_password;Encrypt=True;Connection Timeout=30;",

    "PooledConnection": "Server=localhost;Database=MyApp;User Id=app;Password=pass;Min Pool Size=5;Max Pool Size=100;Connection Lifetime=300;"
  }
}
```

### 주요 연결 옵션

```csharp
// 프로그래밍 방식 연결 문자열 구성
var builder = new SqlConnectionStringBuilder
{
    DataSource = "localhost",
    InitialCatalog = "MyApp",
    IntegratedSecurity = true,
    TrustServerCertificate = true,

    // 풀링 옵션
    Pooling = true,
    MinPoolSize = 5,
    MaxPoolSize = 100,
    ConnectionLifetime = 300,  // 초

    // 타임아웃
    ConnectTimeout = 30,
    CommandTimeout = 30,

    // 복원력
    ConnectRetryCount = 3,
    ConnectRetryInterval = 10
};

string connectionString = builder.ConnectionString;
```

### PostgreSQL 연결 문자열

```json
{
  "ConnectionStrings": {
    "PostgreSQL": "Host=localhost;Database=myapp;Username=user;Password=pass;Pooling=true;Minimum Pool Size=5;Maximum Pool Size=100;"
  }
}
```

---

## 19.2 연결 풀링(Connection Pooling)

### 풀링의 원리

```
연결 풀링 없이:
요청1 → 연결 생성 → 쿼리 → 연결 닫기 → 연결 파괴
요청2 → 연결 생성 → 쿼리 → 연결 닫기 → 연결 파괴
(매번 연결 생성/파괴 오버헤드)

연결 풀링 사용:
                    ┌────────────────┐
요청1 ─┬─ 연결 대여 ─>│   연결 풀      │
요청2 ─┤            │  [연결1]       │
요청3 ─┴─ 연결 반납 <─│  [연결2]       │
                    │  [연결3]       │
                    └────────────────┘
(연결 재사용으로 성능 향상)
```

### 풀링 설정

```csharp
// SQL Server 풀링 옵션
var connectionString = new SqlConnectionStringBuilder
{
    // 기본 연결
    DataSource = "server",
    InitialCatalog = "database",

    // 풀링 활성화 (기본값: true)
    Pooling = true,

    // 최소 풀 크기 (기본값: 0)
    MinPoolSize = 5,

    // 최대 풀 크기 (기본값: 100)
    MaxPoolSize = 100,

    // 연결 수명 (초, 0 = 무제한)
    ConnectionLifetime = 300,

    // 유휴 연결 제거 대기 시간
    // LoadBalanceTimeout = 0 (기본값)
}.ConnectionString;

// PostgreSQL 풀링 옵션 (Npgsql)
var npgsqlBuilder = new NpgsqlConnectionStringBuilder
{
    Host = "localhost",
    Database = "myapp",
    Username = "user",
    Password = "pass",

    Pooling = true,
    MinPoolSize = 5,
    MaxPoolSize = 100,
    ConnectionLifetimeSeconds = 300,
    ConnectionIdleLifetime = 60
};
```

### 풀 모니터링

```csharp
// SQL Server 풀 상태 확인
SqlConnection.ClearPool(connection);     // 특정 연결 풀 초기화
SqlConnection.ClearAllPools();           // 모든 풀 초기화

// 성능 카운터 확인 (Windows)
// perfmon에서 ".NET Data Provider for SqlServer" 카운터 확인
// - NumberOfActiveConnections
// - NumberOfPooledConnections
// - NumberOfFreeConnections
```

---

## 19.3 DbContext 풀링

### 기본 설정

```csharp
// DbContext 풀링 활성화
services.AddDbContextPool<ApplicationDbContext>(options =>
    options.UseSqlServer(connectionString),
    poolSize: 128);  // 기본값: 1024

// 풀링 vs 일반 등록
services.AddDbContext<ApplicationDbContext>(options =>
    options.UseSqlServer(connectionString));
// 매 요청마다 새 인스턴스 생성

services.AddDbContextPool<ApplicationDbContext>(options =>
    options.UseSqlServer(connectionString));
// 인스턴스 재사용
```

### 풀링의 이점

```
벤치마크 결과 (예시):

일반 AddDbContext:
- 인스턴스 생성 시간: ~0.5ms
- 1000 요청 처리: ~500ms

AddDbContextPool:
- 인스턴스 대여 시간: ~0.05ms
- 1000 요청 처리: ~50ms

약 10배 성능 향상!
```

### DbContext 풀링 제약사항

```csharp
// ❌ 풀링 불가: 생성자에서 상태 주입
public class BadContext : DbContext
{
    private readonly string _tenantId;

    public BadContext(
        DbContextOptions<BadContext> options,
        ITenantService tenantService) : base(options)
    {
        _tenantId = tenantService.TenantId;  // 상태 저장!
    }
}

// ✅ 풀링 가능: 상태 없음
public class GoodContext : DbContext
{
    public GoodContext(DbContextOptions<GoodContext> options)
        : base(options)
    {
    }
}

// 테넌트 등 상태가 필요하면 IDbContextFactory 사용
services.AddPooledDbContextFactory<ApplicationDbContext>(options =>
    options.UseSqlServer(connectionString));
```

### IDbContextFactory 사용

```csharp
public class OrderService
{
    private readonly IDbContextFactory<ApplicationDbContext> _contextFactory;

    public OrderService(IDbContextFactory<ApplicationDbContext> contextFactory)
    {
        _contextFactory = contextFactory;
    }

    public async Task ProcessOrderAsync(int orderId)
    {
        // 명시적으로 컨텍스트 생성
        await using var context = await _contextFactory.CreateDbContextAsync();

        var order = await context.Orders.FindAsync(orderId);
        // 작업 수행
    }

    // 장기 실행 작업에 유용
    public async Task ProcessManyOrdersAsync(IEnumerable<int> orderIds)
    {
        foreach (var orderId in orderIds)
        {
            await using var context = await _contextFactory.CreateDbContextAsync();
            // 각 작업마다 새 컨텍스트
        }
    }
}
```

---

## 19.4 연결 복원력(Connection Resiliency)

### 일시적 오류 처리

```csharp
// SQL Server 연결 복원력
services.AddDbContext<ApplicationDbContext>(options =>
    options.UseSqlServer(
        connectionString,
        sqlOptions =>
        {
            sqlOptions.EnableRetryOnFailure(
                maxRetryCount: 5,
                maxRetryDelay: TimeSpan.FromSeconds(30),
                errorNumbersToAdd: null);  // 기본 일시적 오류 사용
        }));

// PostgreSQL 연결 복원력
services.AddDbContext<ApplicationDbContext>(options =>
    options.UseNpgsql(
        connectionString,
        npgsqlOptions =>
        {
            npgsqlOptions.EnableRetryOnFailure(
                maxRetryCount: 5,
                maxRetryDelay: TimeSpan.FromSeconds(30),
                errorCodesToAdd: null);
        }));
```

### 커스텀 실행 전략

```csharp
public class CustomExecutionStrategy : ExecutionStrategy
{
    public CustomExecutionStrategy(
        ExecutionStrategyDependencies dependencies,
        int maxRetryCount,
        TimeSpan maxRetryDelay)
        : base(dependencies, maxRetryCount, maxRetryDelay)
    {
    }

    protected override bool ShouldRetryOn(Exception exception)
    {
        // 재시도할 예외 판단
        if (exception is SqlException sqlException)
        {
            foreach (SqlError error in sqlException.Errors)
            {
                switch (error.Number)
                {
                    case -2:    // 타임아웃
                    case 1205:  // 데드락
                    case 49918: // 리소스 부족
                        return true;
                }
            }
        }

        return false;
    }
}

// 등록
services.AddDbContext<ApplicationDbContext>(options =>
    options.UseSqlServer(
        connectionString,
        sqlOptions =>
        {
            sqlOptions.ExecutionStrategy(deps =>
                new CustomExecutionStrategy(deps, 3, TimeSpan.FromSeconds(10)));
        }));
```

### 트랜잭션과 복원력

```csharp
// 실행 전략과 트랜잭션 함께 사용
var strategy = context.Database.CreateExecutionStrategy();

await strategy.ExecuteAsync(async () =>
{
    using var transaction = await context.Database.BeginTransactionAsync();

    try
    {
        context.Orders.Add(order);
        await context.SaveChangesAsync();

        context.OrderItems.AddRange(items);
        await context.SaveChangesAsync();

        await transaction.CommitAsync();
    }
    catch
    {
        await transaction.RollbackAsync();
        throw;
    }
});
```

---

## 연결 관리 모범 사례

### DO

```csharp
// ✅ DbContext를 짧게 유지
using var context = new ApplicationDbContext();
var data = await context.Products.ToListAsync();
// context 자동 해제

// ✅ 비동기 메서드 사용
await context.Products.ToListAsync();
await context.SaveChangesAsync();

// ✅ 필요한 데이터만 조회
var names = await context.Products
    .Select(p => p.Name)
    .ToListAsync();
```

### DON'T

```csharp
// ❌ 장시간 DbContext 유지
public class LongLivedService
{
    private readonly ApplicationDbContext _context;  // 위험!

    public LongLivedService(ApplicationDbContext context)
    {
        _context = context;  // Scoped를 Singleton처럼 사용
    }
}

// ❌ 연결을 직접 열어두기
var connection = context.Database.GetDbConnection();
await connection.OpenAsync();
// ... 오랜 시간 사용
await connection.CloseAsync();  // 풀에 늦게 반환
```

---

## 요약

| 설정 | 효과 | 권장값 |
|------|------|--------|
| Connection Pooling | 연결 재사용 | 활성화 (기본) |
| Min Pool Size | 최소 연결 유지 | 5-10 |
| Max Pool Size | 최대 연결 제한 | 100 |
| DbContext Pooling | 인스턴스 재사용 | 활성화 권장 |
| Retry on Failure | 일시적 오류 복구 | 활성화 권장 |

## 다음 장 예고

다음 Part에서는 단위 테스트와 통합 테스트 작성 방법을 알아봅니다.
