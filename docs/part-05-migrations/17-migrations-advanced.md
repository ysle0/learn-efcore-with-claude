# Chapter 17: 마이그레이션 고급

## 개요

데이터 시딩, 사용자 정의 마이그레이션, 프로덕션 환경 전략, 다중 DbContext 관리 등 고급 마이그레이션 기법을 알아봅니다.

---

## 16.1 데이터 시딩(Data Seeding)

### 간략 설명
데이터 시딩은 마이그레이션을 통해 초기 데이터를 데이터베이스에 삽입하는 기능입니다.

### HasData를 사용한 시딩

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    // 기본 데이터 시딩
    modelBuilder.Entity<Category>().HasData(
        new Category { Id = 1, Name = "Electronics" },
        new Category { Id = 2, Name = "Books" },
        new Category { Id = 3, Name = "Clothing" }
    );

    // 관계가 있는 데이터
    modelBuilder.Entity<Product>().HasData(
        new Product { Id = 1, Name = "Laptop", CategoryId = 1 },
        new Product { Id = 2, Name = "Phone", CategoryId = 1 },
        new Product { Id = 3, Name = "C# Guide", CategoryId = 2 }
    );

    // 소유 타입 시딩
    modelBuilder.Entity<Customer>().HasData(
        new Customer { Id = 1, Name = "John Doe" }
    );
    modelBuilder.Entity<Customer>().OwnsOne(c => c.Address).HasData(
        new { CustomerId = 1, Street = "123 Main St", City = "Seoul" }
    );
}
```

### 시딩 시 주의사항

```csharp
// ❌ 잘못된 예: ID 미지정
modelBuilder.Entity<Category>().HasData(
    new Category { Name = "Electronics" }  // ID 필수!
);

// ❌ 잘못된 예: 네비게이션 속성 사용
modelBuilder.Entity<Category>().HasData(
    new Category
    {
        Id = 1,
        Name = "Electronics",
        Products = new List<Product>()  // 지원 안 됨!
    }
);

// ✅ 올바른 예: 외래 키 사용
modelBuilder.Entity<Product>().HasData(
    new Product { Id = 1, Name = "Laptop", CategoryId = 1 }
);
```

### 조건부 시딩

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    if (Environment.GetEnvironmentVariable("SEED_DATA") == "true")
    {
        modelBuilder.Entity<Category>().HasData(
            new Category { Id = 1, Name = "Test Category" }
        );
    }
}
```

### 애플리케이션 시작 시 시딩

```csharp
public static class DbInitializer
{
    public static async Task SeedAsync(ApplicationDbContext context)
    {
        // 이미 데이터가 있으면 스킵
        if (await context.Categories.AnyAsync())
            return;

        var categories = new List<Category>
        {
            new Category { Name = "Electronics" },
            new Category { Name = "Books" }
        };

        context.Categories.AddRange(categories);
        await context.SaveChangesAsync();

        var products = new List<Product>
        {
            new Product
            {
                Name = "Laptop",
                Price = 1200,
                Category = categories[0]
            }
        };

        context.Products.AddRange(products);
        await context.SaveChangesAsync();
    }
}

// Program.cs에서 호출
using var scope = app.Services.CreateScope();
var context = scope.ServiceProvider.GetRequiredService<ApplicationDbContext>();
await DbInitializer.SeedAsync(context);
```

---

## 16.2 사용자 정의 마이그레이션

### 마이그레이션 수동 편집

```csharp
public partial class AddProductDescription : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        // 자동 생성된 코드
        migrationBuilder.AddColumn<string>(
            name: "Description",
            table: "Products",
            maxLength: 2000,
            nullable: true);

        // 수동 추가: 기존 데이터 업데이트
        migrationBuilder.Sql(
            "UPDATE Products SET Description = Name + ' description' WHERE Description IS NULL");

        // 수동 추가: 인덱스 생성
        migrationBuilder.CreateIndex(
            name: "IX_Products_Description",
            table: "Products",
            column: "Description");
    }

    protected override void Down(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.DropIndex(
            name: "IX_Products_Description",
            table: "Products");

        migrationBuilder.DropColumn(
            name: "Description",
            table: "Products");
    }
}
```

### Raw SQL 실행

```csharp
protected override void Up(MigrationBuilder migrationBuilder)
{
    // 저장 프로시저 생성
    migrationBuilder.Sql(@"
        CREATE PROCEDURE GetProductsByCategory
            @CategoryId INT
        AS
        BEGIN
            SELECT * FROM Products WHERE CategoryId = @CategoryId
        END");

    // 뷰 생성
    migrationBuilder.Sql(@"
        CREATE VIEW vw_ProductSummary AS
        SELECT
            c.Name AS CategoryName,
            COUNT(p.Id) AS ProductCount,
            AVG(p.Price) AS AveragePrice
        FROM Categories c
        LEFT JOIN Products p ON c.Id = p.CategoryId
        GROUP BY c.Name");

    // 트리거 생성
    migrationBuilder.Sql(@"
        CREATE TRIGGER tr_Products_Audit
        ON Products
        AFTER UPDATE
        AS
        BEGIN
            INSERT INTO AuditLog (TableName, Action, Timestamp)
            VALUES ('Products', 'UPDATE', GETUTCDATE())
        END");
}

protected override void Down(MigrationBuilder migrationBuilder)
{
    migrationBuilder.Sql("DROP TRIGGER IF EXISTS tr_Products_Audit");
    migrationBuilder.Sql("DROP VIEW IF EXISTS vw_ProductSummary");
    migrationBuilder.Sql("DROP PROCEDURE IF EXISTS GetProductsByCategory");
}
```

### 데이터 마이그레이션

```csharp
// 컬럼 분할 예제: FullName → FirstName, LastName
protected override void Up(MigrationBuilder migrationBuilder)
{
    // 1. 새 컬럼 추가
    migrationBuilder.AddColumn<string>(
        name: "FirstName",
        table: "Users",
        nullable: true);

    migrationBuilder.AddColumn<string>(
        name: "LastName",
        table: "Users",
        nullable: true);

    // 2. 데이터 마이그레이션
    migrationBuilder.Sql(@"
        UPDATE Users
        SET FirstName = CASE
                WHEN CHARINDEX(' ', FullName) > 0
                THEN LEFT(FullName, CHARINDEX(' ', FullName) - 1)
                ELSE FullName
            END,
            LastName = CASE
                WHEN CHARINDEX(' ', FullName) > 0
                THEN SUBSTRING(FullName, CHARINDEX(' ', FullName) + 1, LEN(FullName))
                ELSE ''
            END");

    // 3. 컬럼을 NOT NULL로 변경
    migrationBuilder.AlterColumn<string>(
        name: "FirstName",
        table: "Users",
        nullable: false);

    // 4. 이전 컬럼 삭제
    migrationBuilder.DropColumn(
        name: "FullName",
        table: "Users");
}
```

---

## 16.3 프로덕션 환경 마이그레이션 전략

### 안전한 배포 전략

```
권장 배포 프로세스:

1. 마이그레이션 스크립트 생성
   dotnet ef migrations script --idempotent -o deploy.sql

2. 스크립트 검토
   - DBA 또는 팀원 리뷰
   - 데이터 손실 여부 확인

3. 테스트 환경에서 실행
   - 프로덕션 데이터 복사본으로 테스트

4. 백업
   - 프로덕션 데이터베이스 백업

5. 유지보수 시간에 배포
   - 트래픽이 적은 시간 선택

6. 마이그레이션 실행
   - 스크립트 또는 번들 실행

7. 애플리케이션 배포
```

### 다운타임 최소화

```csharp
// ❌ 잠금을 유발하는 작업
// 대용량 테이블에 NOT NULL 컬럼 추가
migrationBuilder.AddColumn<string>(
    name: "NewColumn",
    table: "LargeTable",
    nullable: false,
    defaultValue: "");  // 모든 행 업데이트 필요

// ✅ 단계적 마이그레이션
// 1단계: nullable 컬럼 추가 (빠름)
migrationBuilder.AddColumn<string>(
    name: "NewColumn",
    table: "LargeTable",
    nullable: true);

// 2단계: 배치로 데이터 업데이트 (별도 스크립트)
// UPDATE TOP (10000) LargeTable SET NewColumn = 'default' WHERE NewColumn IS NULL

// 3단계: NOT NULL로 변경 (데이터 채워진 후)
migrationBuilder.AlterColumn<string>(
    name: "NewColumn",
    table: "LargeTable",
    nullable: false);
```

### 롤백 계획

```csharp
// 모든 마이그레이션에 Down 메서드 구현
protected override void Down(MigrationBuilder migrationBuilder)
{
    // 되돌리기 로직 구현
    // 데이터 손실 가능성 문서화
}

// 롤백 스크립트 미리 준비
// dotnet ef migrations script AddNewFeature InitialCreate -o rollback.sql
```

---

## 16.4 여러 DbContext 관리

### 다중 컨텍스트 설정

```csharp
// 주문 컨텍스트
public class OrderDbContext : DbContext
{
    public DbSet<Order> Orders => Set<Order>();
}

// 재고 컨텍스트
public class InventoryDbContext : DbContext
{
    public DbSet<Product> Products => Set<Product>();
}

// 마이그레이션 생성 시 컨텍스트 지정
// dotnet ef migrations add Initial --context OrderDbContext
// dotnet ef migrations add Initial --context InventoryDbContext

// 별도 폴더로 분리
// dotnet ef migrations add Initial --context OrderDbContext --output-dir Migrations/Orders
// dotnet ef migrations add Initial --context InventoryDbContext --output-dir Migrations/Inventory
```

### 마이그레이션 적용

```bash
# 특정 컨텍스트 업데이트
dotnet ef database update --context OrderDbContext
dotnet ef database update --context InventoryDbContext

# 프로그래밍 방식
await orderContext.Database.MigrateAsync();
await inventoryContext.Database.MigrateAsync();
```

### 공유 마이그레이션 히스토리

```csharp
// 같은 데이터베이스, 다른 스키마
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
{
    optionsBuilder.UseSqlServer(connectionString, options =>
    {
        options.MigrationsHistoryTable("__EFMigrationsHistory", "orders");
    });
}
```

---

## 마이그레이션 문제 해결

### 흔한 문제들

```bash
# 모델과 스냅샷 불일치
# 해결: 스냅샷 재생성
dotnet ef migrations remove
dotnet ef migrations add <Name>

# 빈 마이그레이션 생성됨
# 원인: 모델 변경이 없음
# 해결: 불필요한 마이그레이션 제거

# 마이그레이션 순서 충돌 (팀 작업)
# 해결: 마이그레이션 병합 또는 재생성
```

### 강제 재생성

```bash
# 모든 마이그레이션 삭제 후 재생성
rm -rf Migrations/

# 히스토리 테이블 삭제 (주의!)
# DELETE FROM __EFMigrationsHistory

# 새 마이그레이션 생성
dotnet ef migrations add InitialCreate
```

---

## 요약

| 기능 | 설명 | 사용 시나리오 |
|------|------|-------------|
| HasData | 정적 시드 데이터 | 참조 데이터, 기본 설정 |
| Sql() | Raw SQL 실행 | SP, 뷰, 데이터 변환 |
| 번들 | 실행 파일 | CI/CD, 프로덕션 배포 |
| 멀티 컨텍스트 | 분리된 컨텍스트 | 마이크로서비스, 모듈화 |

## 다음 장 예고

다음 Part에서는 쿼리 성능 최적화, 벌크 작업, 연결 풀링 등 성능 관련 주제를 다룹니다.
