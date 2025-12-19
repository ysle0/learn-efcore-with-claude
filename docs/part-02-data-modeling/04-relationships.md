# Chapter 04: 관계(Relationships) 설정

## 개요

엔티티 간의 관계를 설정하는 방법을 알아봅니다. 일대다, 일대일, 다대다 관계의 구성과 네비게이션 속성, 외래 키 설정 방법을 상세히 다룹니다.

---

## 4.1 일대다(One-to-Many) 관계

### 간략 설명
가장 흔한 관계 유형으로, 하나의 엔티티가 여러 개의 다른 엔티티와 연결됩니다. 예: 하나의 블로그에 여러 포스트가 속함.

### 상세 설명

#### 기본 구조

```csharp
// 주체(Principal) 엔티티
public class Blog
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;

    // 컬렉션 네비게이션 속성
    public ICollection<Post> Posts { get; set; } = new List<Post>();
}

// 종속(Dependent) 엔티티
public class Post
{
    public int Id { get; set; }
    public string Title { get; set; } = string.Empty;
    public string Content { get; set; } = string.Empty;

    // 외래 키 속성
    public int BlogId { get; set; }

    // 참조 네비게이션 속성
    public Blog Blog { get; set; } = null!;
}
```

#### 생성되는 테이블

```sql
CREATE TABLE Blogs (
    Id int NOT NULL IDENTITY PRIMARY KEY,
    Name nvarchar(max) NOT NULL
);

CREATE TABLE Posts (
    Id int NOT NULL IDENTITY PRIMARY KEY,
    Title nvarchar(max) NOT NULL,
    Content nvarchar(max) NOT NULL,
    BlogId int NOT NULL,
    CONSTRAINT FK_Posts_Blogs_BlogId
        FOREIGN KEY (BlogId) REFERENCES Blogs(Id) ON DELETE CASCADE
);

CREATE INDEX IX_Posts_BlogId ON Posts (BlogId);
```

#### 관계 다이어그램

```
┌─────────────────┐         ┌─────────────────┐
│      Blog       │         │      Post       │
├─────────────────┤         ├─────────────────┤
│ PK  Id          │────┐    │ PK  Id          │
│     Name        │    │    │     Title       │
│                 │    │    │     Content     │
│                 │    └───→│ FK  BlogId      │
└─────────────────┘         └─────────────────┘
        1                           *
```

#### 다양한 구성 방식

```csharp
// 1. 컨벤션 기반 (자동 감지)
public class Blog
{
    public int Id { get; set; }
    public ICollection<Post> Posts { get; set; } = new List<Post>();
}

public class Post
{
    public int Id { get; set; }
    public int BlogId { get; set; }  // 컨벤션: {Navigation}Id
    public Blog Blog { get; set; } = null!;
}

// 2. 외래 키만 있는 경우
public class Post
{
    public int Id { get; set; }
    public int BlogId { get; set; }  // 네비게이션 없이 FK만
}

// 3. 네비게이션만 있는 경우 (Shadow FK)
public class Post
{
    public int Id { get; set; }
    public Blog Blog { get; set; } = null!;  // FK는 Shadow Property로
}

// 4. 양방향 네비게이션
public class Blog
{
    public ICollection<Post> Posts { get; set; } = new List<Post>();
}
public class Post
{
    public Blog Blog { get; set; } = null!;
}
```

#### Fluent API 구성

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    // 기본 구성
    modelBuilder.Entity<Post>()
        .HasOne(p => p.Blog)
        .WithMany(b => b.Posts)
        .HasForeignKey(p => p.BlogId);

    // 삭제 동작 설정
    modelBuilder.Entity<Post>()
        .HasOne(p => p.Blog)
        .WithMany(b => b.Posts)
        .HasForeignKey(p => p.BlogId)
        .OnDelete(DeleteBehavior.Cascade);  // 기본값

    // 필수 관계 vs 선택적 관계
    modelBuilder.Entity<Post>()
        .HasOne(p => p.Blog)
        .WithMany(b => b.Posts)
        .HasForeignKey(p => p.BlogId)
        .IsRequired();  // NOT NULL 외래 키

    // 선택적 관계 (nullable 외래 키)
    modelBuilder.Entity<Post>()
        .HasOne(p => p.Blog)
        .WithMany(b => b.Posts)
        .HasForeignKey(p => p.BlogId)
        .IsRequired(false);  // NULL 허용
}
```

#### 삭제 동작 옵션

```csharp
// DeleteBehavior 옵션

// 1. Cascade (기본값 - 필수 관계)
// 주체 삭제 시 종속도 함께 삭제
.OnDelete(DeleteBehavior.Cascade)

// 2. Restrict
// 종속이 있으면 주체 삭제 불가 (예외 발생)
.OnDelete(DeleteBehavior.Restrict)

// 3. SetNull (선택적 관계에서만)
// 주체 삭제 시 외래 키를 NULL로 설정
.OnDelete(DeleteBehavior.SetNull)

// 4. NoAction
// 데이터베이스가 처리 (일반적으로 오류)
.OnDelete(DeleteBehavior.NoAction)

// 5. ClientSetNull (기본값 - 선택적 관계)
// 추적 중인 엔티티의 FK를 NULL로, DB는 NoAction
.OnDelete(DeleteBehavior.ClientSetNull)

// 6. ClientCascade
// 추적 중인 엔티티 삭제, DB는 NoAction
.OnDelete(DeleteBehavior.ClientCascade)
```

---

## 4.2 일대일(One-to-One) 관계

### 간략 설명
두 엔티티가 정확히 하나씩 서로 연결됩니다. 예: 사용자와 사용자 프로필.

### 상세 설명

#### 기본 구조

```csharp
// 주체 엔티티
public class User
{
    public int Id { get; set; }
    public string Username { get; set; } = string.Empty;

    // 참조 네비게이션
    public UserProfile? Profile { get; set; }
}

// 종속 엔티티
public class UserProfile
{
    public int Id { get; set; }
    public string Bio { get; set; } = string.Empty;
    public string? AvatarUrl { get; set; }

    // 외래 키 (유니크 제약)
    public int UserId { get; set; }

    // 참조 네비게이션
    public User User { get; set; } = null!;
}
```

#### 생성되는 테이블

```sql
CREATE TABLE Users (
    Id int NOT NULL IDENTITY PRIMARY KEY,
    Username nvarchar(max) NOT NULL
);

CREATE TABLE UserProfiles (
    Id int NOT NULL IDENTITY PRIMARY KEY,
    Bio nvarchar(max) NOT NULL,
    AvatarUrl nvarchar(max) NULL,
    UserId int NOT NULL UNIQUE,  -- 유니크 제약
    CONSTRAINT FK_UserProfiles_Users_UserId
        FOREIGN KEY (UserId) REFERENCES Users(Id) ON DELETE CASCADE
);
```

#### Fluent API 구성

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<User>()
        .HasOne(u => u.Profile)
        .WithOne(p => p.User)
        .HasForeignKey<UserProfile>(p => p.UserId);  // 종속 엔티티 명시
}
```

#### 공유 기본 키 패턴

```csharp
// 종속 엔티티가 주체의 키를 기본 키로 사용
public class User
{
    public int Id { get; set; }
    public string Username { get; set; } = string.Empty;
    public UserProfile? Profile { get; set; }
}

public class UserProfile
{
    public int Id { get; set; }  // User.Id와 동일한 값
    public string Bio { get; set; } = string.Empty;
    public User User { get; set; } = null!;
}

// Fluent API
modelBuilder.Entity<UserProfile>()
    .HasOne(p => p.User)
    .WithOne(u => u.Profile)
    .HasForeignKey<UserProfile>(p => p.Id);  // PK가 FK

// 장점: 조인 시 성능 향상, 저장 공간 절약
```

#### 선택적 일대일 관계

```csharp
public class Order
{
    public int Id { get; set; }
    public string OrderNumber { get; set; } = string.Empty;

    // 선택적 관계 - 배송되지 않은 주문도 있음
    public Shipment? Shipment { get; set; }
}

public class Shipment
{
    public int Id { get; set; }
    public string TrackingNumber { get; set; } = string.Empty;

    public int OrderId { get; set; }
    public Order Order { get; set; } = null!;
}

modelBuilder.Entity<Order>()
    .HasOne(o => o.Shipment)
    .WithOne(s => s.Order)
    .HasForeignKey<Shipment>(s => s.OrderId)
    .IsRequired(false);  // 선택적
```

---

## 4.3 다대다(Many-to-Many) 관계

### 간략 설명
두 엔티티가 서로 여러 개씩 연결됩니다. 예: 학생들과 수업들.

### EF Core 5+ 간소화된 다대다

```csharp
// 조인 엔티티 없이 직접 구성
public class Student
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;

    public ICollection<Course> Courses { get; set; } = new List<Course>();
}

public class Course
{
    public int Id { get; set; }
    public string Title { get; set; } = string.Empty;

    public ICollection<Student> Students { get; set; } = new List<Student>();
}

// EF Core가 자동으로 조인 테이블 생성
```

#### 자동 생성되는 조인 테이블

```sql
CREATE TABLE CourseStudent (
    CoursesId int NOT NULL,
    StudentsId int NOT NULL,
    PRIMARY KEY (CoursesId, StudentsId),
    FOREIGN KEY (CoursesId) REFERENCES Courses(Id) ON DELETE CASCADE,
    FOREIGN KEY (StudentsId) REFERENCES Students(Id) ON DELETE CASCADE
);
```

#### Fluent API로 조인 테이블 커스터마이징

```csharp
modelBuilder.Entity<Student>()
    .HasMany(s => s.Courses)
    .WithMany(c => c.Students)
    .UsingEntity(j => j.ToTable("Enrollments"));  // 테이블 이름 변경

// 더 상세한 커스터마이징
modelBuilder.Entity<Student>()
    .HasMany(s => s.Courses)
    .WithMany(c => c.Students)
    .UsingEntity<Dictionary<string, object>>(
        "Enrollment",
        j => j.HasOne<Course>().WithMany().HasForeignKey("CourseId"),
        j => j.HasOne<Student>().WithMany().HasForeignKey("StudentId"),
        j =>
        {
            j.HasKey("StudentId", "CourseId");
            j.ToTable("Enrollments");
        });
```

### 명시적 조인 엔티티 (추가 데이터 필요 시)

```csharp
// 조인 테이블에 추가 속성이 필요한 경우
public class Student
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;

    public ICollection<Enrollment> Enrollments { get; set; } = new List<Enrollment>();
}

public class Course
{
    public int Id { get; set; }
    public string Title { get; set; } = string.Empty;

    public ICollection<Enrollment> Enrollments { get; set; } = new List<Enrollment>();
}

// 조인 엔티티
public class Enrollment
{
    public int StudentId { get; set; }
    public int CourseId { get; set; }

    // 추가 속성들
    public DateTime EnrolledAt { get; set; }
    public Grade? Grade { get; set; }
    public bool IsCompleted { get; set; }

    // 네비게이션
    public Student Student { get; set; } = null!;
    public Course Course { get; set; } = null!;
}

public enum Grade { A, B, C, D, F }

// Fluent API 구성
modelBuilder.Entity<Enrollment>(entity =>
{
    entity.HasKey(e => new { e.StudentId, e.CourseId });

    entity.HasOne(e => e.Student)
        .WithMany(s => s.Enrollments)
        .HasForeignKey(e => e.StudentId);

    entity.HasOne(e => e.Course)
        .WithMany(c => c.Enrollments)
        .HasForeignKey(e => e.CourseId);

    entity.Property(e => e.EnrolledAt)
        .HasDefaultValueSql("GETUTCDATE()");
});
```

### Skip Navigation (EF Core 5+)

```csharp
// 조인 엔티티가 있어도 직접 접근 가능
public class Student
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;

    // Skip Navigation - 조인 엔티티 건너뛰기
    public ICollection<Course> Courses { get; set; } = new List<Course>();

    // 조인 엔티티 접근도 가능
    public ICollection<Enrollment> Enrollments { get; set; } = new List<Enrollment>();
}

public class Course
{
    public int Id { get; set; }
    public string Title { get; set; } = string.Empty;

    public ICollection<Student> Students { get; set; } = new List<Student>();
    public ICollection<Enrollment> Enrollments { get; set; } = new List<Enrollment>();
}

// Fluent API
modelBuilder.Entity<Student>()
    .HasMany(s => s.Courses)
    .WithMany(c => c.Students)
    .UsingEntity<Enrollment>(
        l => l.HasOne(e => e.Course).WithMany(c => c.Enrollments),
        r => r.HasOne(e => e.Student).WithMany(s => s.Enrollments));

// 사용 예시
var student = await context.Students
    .Include(s => s.Courses)  // Skip Navigation 사용
    .FirstAsync();

foreach (var course in student.Courses)
{
    Console.WriteLine(course.Title);
}
```

---

## 4.4 자기 참조 관계(Self-Referencing)

### 간략 설명
같은 엔티티 타입 내에서 관계를 맺는 패턴입니다. 예: 조직도의 상사-부하 관계, 카테고리의 상위-하위 관계.

### 일대다 자기 참조

```csharp
// 카테고리 계층 구조
public class Category
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;

    // 부모 참조 (nullable - 루트 카테고리)
    public int? ParentCategoryId { get; set; }
    public Category? ParentCategory { get; set; }

    // 자식 컬렉션
    public ICollection<Category> SubCategories { get; set; } = new List<Category>();
}

// Fluent API
modelBuilder.Entity<Category>()
    .HasOne(c => c.ParentCategory)
    .WithMany(c => c.SubCategories)
    .HasForeignKey(c => c.ParentCategoryId)
    .OnDelete(DeleteBehavior.Restrict);  // 순환 참조 방지
```

### 다대다 자기 참조

```csharp
// 소셜 네트워크 팔로우 관계
public class User
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;

    // 내가 팔로우하는 사람들
    public ICollection<User> Following { get; set; } = new List<User>();

    // 나를 팔로우하는 사람들
    public ICollection<User> Followers { get; set; } = new List<User>();
}

// Fluent API
modelBuilder.Entity<User>()
    .HasMany(u => u.Following)
    .WithMany(u => u.Followers)
    .UsingEntity<Dictionary<string, object>>(
        "UserFollows",
        j => j.HasOne<User>().WithMany().HasForeignKey("FollowingId"),
        j => j.HasOne<User>().WithMany().HasForeignKey("FollowerId"));

// 또는 명시적 조인 엔티티
public class UserFollow
{
    public int FollowerId { get; set; }
    public int FollowingId { get; set; }
    public DateTime FollowedAt { get; set; }

    public User Follower { get; set; } = null!;
    public User Following { get; set; } = null!;
}
```

### 조직도 예제

```csharp
public class Employee
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public string Title { get; set; } = string.Empty;

    public int? ManagerId { get; set; }
    public Employee? Manager { get; set; }
    public ICollection<Employee> DirectReports { get; set; } = new List<Employee>();
}

// 계층 구조 쿼리
// 특정 직원의 모든 상위 관리자 조회
var employee = await context.Employees
    .Include(e => e.Manager)
        .ThenInclude(m => m!.Manager)
            .ThenInclude(m => m!.Manager)
    .FirstOrDefaultAsync(e => e.Id == employeeId);

// 재귀 CTE를 사용한 전체 계층 조회 (Raw SQL)
var hierarchy = await context.Employees
    .FromSqlRaw(@"
        WITH EmployeeHierarchy AS (
            SELECT Id, Name, Title, ManagerId, 0 as Level
            FROM Employees
            WHERE Id = {0}
            UNION ALL
            SELECT e.Id, e.Name, e.Title, e.ManagerId, eh.Level + 1
            FROM Employees e
            JOIN EmployeeHierarchy eh ON e.ManagerId = eh.Id
        )
        SELECT * FROM EmployeeHierarchy", employeeId)
    .ToListAsync();
```

---

## 관계 설정 베스트 프랙티스

### 1. 네비게이션 속성 초기화

```csharp
// ✅ 좋은 예: 컬렉션 초기화
public class Blog
{
    public int Id { get; set; }
    public ICollection<Post> Posts { get; set; } = new List<Post>();
}

// ❌ 나쁜 예: null 가능
public class Blog
{
    public int Id { get; set; }
    public ICollection<Post> Posts { get; set; }  // NullReferenceException 위험
}

// ✅ 좋은 예: null-forgiving 연산자 사용
public class Post
{
    public int Id { get; set; }
    public Blog Blog { get; set; } = null!;  // EF Core가 채워줌
}
```

### 2. 캡슐화된 컬렉션

```csharp
// 도메인 로직이 있는 경우
public class Order
{
    public int Id { get; set; }

    private readonly List<OrderItem> _items = new();
    public IReadOnlyCollection<OrderItem> Items => _items.AsReadOnly();

    public void AddItem(Product product, int quantity)
    {
        if (quantity <= 0)
            throw new ArgumentException("수량은 0보다 커야 합니다.");

        var existingItem = _items.FirstOrDefault(i => i.ProductId == product.Id);
        if (existingItem != null)
        {
            existingItem.IncreaseQuantity(quantity);
        }
        else
        {
            _items.Add(new OrderItem(this, product, quantity));
        }
    }
}

// Fluent API에서 백킹 필드 지정
modelBuilder.Entity<Order>()
    .HasMany(o => o.Items)
    .WithOne(i => i.Order)
    .HasForeignKey(i => i.OrderId);

modelBuilder.Entity<Order>()
    .Navigation(o => o.Items)
    .HasField("_items")
    .UsePropertyAccessMode(PropertyAccessMode.Field);
```

### 3. 외래 키 명시적 정의

```csharp
// ✅ 명시적 외래 키 (권장)
public class Post
{
    public int Id { get; set; }
    public int BlogId { get; set; }  // 명시적
    public Blog Blog { get; set; } = null!;
}

// 장점:
// - 네비게이션 로드 없이 FK 접근 가능
// - 관계가 명확함
// - 성능 향상 (불필요한 로드 방지)
```

---

## 요약

| 관계 유형 | 주요 특징 | FK 위치 |
|----------|----------|---------|
| 일대다 | 가장 흔함, 컬렉션 네비게이션 | 다(Many) 쪽 |
| 일대일 | 유니크 FK, 명시적 종속 지정 필요 | 종속 엔티티 |
| 다대다 | 조인 테이블 필요, EF Core 5+에서 간소화 | 조인 테이블 |
| 자기 참조 | 동일 엔티티 내 관계, nullable FK 주의 | 자기 자신 |

## 다음 장 예고

다음 장에서는 Fluent API를 더 깊이 살펴보고, 엔티티 구성 분리와 고급 인덱스 설정 방법을 알아봅니다.
