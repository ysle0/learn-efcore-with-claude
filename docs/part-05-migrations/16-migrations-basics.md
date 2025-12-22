# Chapter 16: 마이그레이션 기초

## 개요

EF Core 마이그레이션을 사용하여 데이터베이스 스키마를 버전 관리하는 방법을 알아봅니다. 마이그레이션 생성, 적용, 롤백 방법을 다룹니다.

---

## 15.1 마이그레이션이란?

### 간략 설명
마이그레이션은 C# 코드로 작성된 데이터베이스 스키마 변경 기록입니다. 모델 변경을 데이터베이스에 점진적으로 적용할 수 있습니다.

### 마이그레이션의 필요성

```
개발 워크플로우:

1. 모델 변경 (C# 코드)
   └─> 엔티티 속성 추가, 관계 변경 등

2. 마이그레이션 생성
   └─> dotnet ef migrations add <Name>

3. 마이그레이션 적용
   └─> dotnet ef database update

4. 반복
```

### 마이그레이션 파일 구조

```
Migrations/
├── 20240101120000_InitialCreate.cs           # Up/Down 메서드
├── 20240101120000_InitialCreate.Designer.cs  # 모델 스냅샷
├── 20240115150000_AddProductDescription.cs
├── 20240115150000_AddProductDescription.Designer.cs
└── ApplicationDbContextModelSnapshot.cs       # 현재 모델 상태
```

---

## 15.2 Add-Migration, Update-Database

### 마이그레이션 생성

```bash
# .NET CLI
dotnet ef migrations add InitialCreate

# 프로젝트 지정
dotnet ef migrations add InitialCreate --project Data --startup-project Web

# 출력 폴더 지정
dotnet ef migrations add InitialCreate --output-dir Data/Migrations

# Package Manager Console (Visual Studio)
Add-Migration InitialCreate
```

### 생성되는 파일

```csharp
// 20240101120000_InitialCreate.cs
public partial class InitialCreate : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.CreateTable(
            name: "Products",
            columns: table => new
            {
                Id = table.Column<int>(nullable: false)
                    .Annotation("SqlServer:Identity", "1, 1"),
                Name = table.Column<string>(maxLength: 200, nullable: false),
                Price = table.Column<decimal>(type: "decimal(18,2)", nullable: false)
            },
            constraints: table =>
            {
                table.PrimaryKey("PK_Products", x => x.Id);
            });
    }

    protected override void Down(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.DropTable(name: "Products");
    }
}
```

### 마이그레이션 적용

```bash
# 최신 마이그레이션까지 적용
dotnet ef database update

# 특정 마이그레이션까지 적용
dotnet ef database update InitialCreate

# 연결 문자열 직접 지정
dotnet ef database update --connection "Server=...;Database=..."

# Package Manager Console
Update-Database
Update-Database InitialCreate
```

### 마이그레이션 목록

```bash
# 모든 마이그레이션 표시
dotnet ef migrations list

# 출력 예시:
# 20240101120000_InitialCreate
# 20240115150000_AddProductDescription (Pending)

# Package Manager Console
Get-Migration
```

---

## 15.3 마이그레이션 되돌리기

### 이전 마이그레이션으로 롤백

```bash
# 특정 마이그레이션으로 롤백
dotnet ef database update AddProductDescription

# 모든 마이그레이션 롤백 (빈 데이터베이스)
dotnet ef database update 0

# Package Manager Console
Update-Database AddProductDescription
Update-Database 0
```

### 마이그레이션 제거

```bash
# 마지막 마이그레이션 제거 (적용 전)
dotnet ef migrations remove

# 강제 제거 (적용 후에도)
dotnet ef migrations remove --force

# Package Manager Console
Remove-Migration
Remove-Migration -Force
```

### 롤백 시 주의사항

```csharp
// ❌ 데이터 손실 위험
protected override void Down(MigrationBuilder migrationBuilder)
{
    migrationBuilder.DropColumn(
        name: "Description",
        table: "Products");
    // Description 컬럼의 데이터가 삭제됨
}

// ✅ 데이터 보존 전략
// 1. 롤백 전 데이터 백업
// 2. 컬럼 삭제 대신 nullable로 변경 고려
// 3. 운영 환경에서는 롤백보다 새 마이그레이션 권장
```

---

## 15.4 마이그레이션 스크립트 생성

### SQL 스크립트 생성

```bash
# 모든 마이그레이션 스크립트
dotnet ef migrations script

# 특정 범위
dotnet ef migrations script InitialCreate AddProductDescription

# 파일로 저장
dotnet ef migrations script -o migrations.sql

# 멱등성 스크립트 (여러 번 실행해도 안전)
dotnet ef migrations script --idempotent

# Package Manager Console
Script-Migration
Script-Migration -From InitialCreate -To AddProductDescription
Script-Migration -Idempotent -Output migrations.sql
```

### 멱등성 스크립트

```sql
-- 멱등성 스크립트 예시
IF NOT EXISTS (
    SELECT * FROM [__EFMigrationsHistory]
    WHERE [MigrationId] = N'20240101120000_InitialCreate'
)
BEGIN
    CREATE TABLE [Products] (
        [Id] int NOT NULL IDENTITY,
        [Name] nvarchar(200) NOT NULL,
        CONSTRAINT [PK_Products] PRIMARY KEY ([Id])
    );

    INSERT INTO [__EFMigrationsHistory] ([MigrationId], [ProductVersion])
    VALUES (N'20240101120000_InitialCreate', N'8.0.0');
END;
GO
```

### 번들 생성 (EF Core 6+)

```bash
# 마이그레이션 번들 (실행 파일)
dotnet ef migrations bundle

# 자체 포함 번들
dotnet ef migrations bundle --self-contained

# 실행
./efbundle --connection "Server=...;Database=..."
```

---

## 마이그레이션 히스토리 테이블

### __EFMigrationsHistory

```sql
-- 자동 생성되는 테이블
CREATE TABLE [__EFMigrationsHistory] (
    [MigrationId] nvarchar(150) NOT NULL,
    [ProductVersion] nvarchar(32) NOT NULL,
    CONSTRAINT [PK___EFMigrationsHistory] PRIMARY KEY ([MigrationId])
);

-- 적용된 마이그레이션 기록
-- MigrationId                          ProductVersion
-- 20240101120000_InitialCreate         8.0.0
-- 20240115150000_AddProductDescription 8.0.0
```

### 커스텀 히스토리 테이블

```csharp
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
{
    optionsBuilder.UseSqlServer(
        connectionString,
        options => options.MigrationsHistoryTable(
            tableName: "MigrationHistory",
            schema: "admin"));
}
```

---

## 프로그래밍 방식 마이그레이션

### 코드에서 마이그레이션 적용

```csharp
// 애플리케이션 시작 시 마이그레이션 적용
public class Program
{
    public static async Task Main(string[] args)
    {
        var host = CreateHostBuilder(args).Build();

        using (var scope = host.Services.CreateScope())
        {
            var context = scope.ServiceProvider
                .GetRequiredService<ApplicationDbContext>();

            // 대기 중인 마이그레이션 적용
            await context.Database.MigrateAsync();
        }

        await host.RunAsync();
    }
}

// 또는 EnsureCreated (마이그레이션 없이 생성)
await context.Database.EnsureCreatedAsync();
// 주의: 기존 마이그레이션과 호환되지 않음
```

### 대기 중인 마이그레이션 확인

```csharp
// 대기 중인 마이그레이션
var pending = await context.Database.GetPendingMigrationsAsync();

// 적용된 마이그레이션
var applied = await context.Database.GetAppliedMigrationsAsync();

// 마이그레이션 적용 여부
bool hasPending = pending.Any();
```

---

## 요약

| 명령어 | 용도 | 주요 옵션 |
|--------|------|----------|
| `migrations add` | 마이그레이션 생성 | --output-dir |
| `database update` | 마이그레이션 적용 | --connection |
| `migrations remove` | 마이그레이션 제거 | --force |
| `migrations list` | 목록 조회 | - |
| `migrations script` | SQL 스크립트 | --idempotent |
| `migrations bundle` | 실행 파일 | --self-contained |

## 다음 장 예고

다음 장에서는 데이터 시딩, 사용자 정의 마이그레이션 등 고급 마이그레이션 기법을 알아봅니다.
