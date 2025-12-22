# Chapter 08: 상속 매핑 전략

## 개요

객체 지향 프로그래밍의 상속을 관계형 데이터베이스에 매핑하는 세 가지 전략을 알아봅니다. TPH, TPT, TPC 각각의 특징과 사용 시나리오를 상세히 다룹니다.

---

## 상속 매핑 개념

### 왜 상속 매핑이 필요한가?

```csharp
// 객체 지향 모델
public abstract class Payment
{
    public int Id { get; set; }
    public decimal Amount { get; set; }
    public DateTime PaymentDate { get; set; }
}

public class CreditCardPayment : Payment
{
    public string CardNumber { get; set; } = string.Empty;
    public string CardHolderName { get; set; } = string.Empty;
}

public class BankTransferPayment : Payment
{
    public string BankName { get; set; } = string.Empty;
    public string AccountNumber { get; set; } = string.Empty;
}

public class CryptoPayment : Payment
{
    public string WalletAddress { get; set; } = string.Empty;
    public string CryptoCurrency { get; set; } = string.Empty;
}
```

**문제**: 관계형 데이터베이스는 상속을 직접 지원하지 않음

**해결책**: 세 가지 매핑 전략

---

## 6.1 TPH (Table Per Hierarchy)

### 간략 설명
전체 상속 계층을 하나의 테이블에 저장합니다. 구분자(Discriminator) 컬럼으로 타입을 구분합니다.

### 상세 설명

#### 기본 구성

```csharp
// 기본 클래스
public abstract class Payment
{
    public int Id { get; set; }
    public decimal Amount { get; set; }
    public DateTime PaymentDate { get; set; }
}

// 파생 클래스들
public class CreditCardPayment : Payment
{
    public string CardNumber { get; set; } = string.Empty;
    public string CardHolderName { get; set; } = string.Empty;
}

public class BankTransferPayment : Payment
{
    public string BankName { get; set; } = string.Empty;
    public string AccountNumber { get; set; } = string.Empty;
}

// DbContext
public class PaymentContext : DbContext
{
    public DbSet<Payment> Payments => Set<Payment>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // TPH는 기본 전략 - 별도 설정 불필요
        modelBuilder.Entity<Payment>()
            .HasDiscriminator<string>("PaymentType")
            .HasValue<CreditCardPayment>("CreditCard")
            .HasValue<BankTransferPayment>("BankTransfer");
    }
}
```

#### 생성되는 테이블

```sql
CREATE TABLE Payments (
    Id int NOT NULL IDENTITY PRIMARY KEY,
    PaymentType nvarchar(21) NOT NULL,  -- 구분자
    Amount decimal(18,2) NOT NULL,
    PaymentDate datetime2 NOT NULL,

    -- CreditCardPayment 전용
    CardNumber nvarchar(max) NULL,
    CardHolderName nvarchar(max) NULL,

    -- BankTransferPayment 전용
    BankName nvarchar(max) NULL,
    AccountNumber nvarchar(max) NULL
);
```

#### 테이블 구조

```
┌─────────────────────────────────────────────────────────────────────┐
│                           Payments                                   │
├────┬─────────────┬────────┬─────────────┬──────────┬───────────────┤
│ Id │ PaymentType │ Amount │ PaymentDate │ CardNum  │ BankName      │
├────┼─────────────┼────────┼─────────────┼──────────┼───────────────┤
│ 1  │ CreditCard  │ 100.00 │ 2024-01-01  │ 4111...  │ NULL          │
│ 2  │ BankTransfer│ 200.00 │ 2024-01-02  │ NULL     │ 국민은행       │
│ 3  │ CreditCard  │ 150.00 │ 2024-01-03  │ 5500...  │ NULL          │
└────┴─────────────┴────────┴─────────────┴──────────┴───────────────┘
```

#### Fluent API 구성

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Payment>(entity =>
    {
        // 테이블 이름
        entity.ToTable("Payments");

        // 구분자 설정
        entity.HasDiscriminator<string>("PaymentType")
            .HasValue<Payment>("Base")  // 추상 클래스도 값 지정 가능
            .HasValue<CreditCardPayment>("CC")
            .HasValue<BankTransferPayment>("BT");

        // 구분자 컬럼 구성
        entity.Property("PaymentType")
            .HasMaxLength(10)
            .IsRequired();
    });

    // 파생 클래스 속성 구성
    modelBuilder.Entity<CreditCardPayment>(entity =>
    {
        entity.Property(c => c.CardNumber)
            .HasMaxLength(20);
    });

    modelBuilder.Entity<BankTransferPayment>(entity =>
    {
        entity.Property(b => b.AccountNumber)
            .HasMaxLength(30);
    });
}
```

#### 쿼리 예제

```csharp
// 모든 결제 조회
var allPayments = await context.Payments.ToListAsync();

// 특정 타입만 조회
var creditCardPayments = await context.Payments
    .OfType<CreditCardPayment>()
    .ToListAsync();

// 조건부 조회
var recentPayments = await context.Payments
    .Where(p => p.PaymentDate > DateTime.UtcNow.AddDays(-30))
    .ToListAsync();

// 생성되는 SQL
// SELECT * FROM Payments WHERE PaymentType IN ('CreditCard', 'BankTransfer')

// OfType 사용 시
// SELECT * FROM Payments WHERE PaymentType = 'CreditCard'
```

### TPH 장단점

| 장점 | 단점 |
|------|------|
| 단일 테이블로 쿼리 단순 | NULL 컬럼 많음 |
| JOIN 없이 전체 조회 가능 | 데이터 무결성 약함 |
| 마이그레이션 간단 | 특정 타입 제약 어려움 |
| 가장 빠른 성능 | 테이블이 커질 수 있음 |

---

## 6.2 TPT (Table Per Type)

### 간략 설명
각 타입마다 별도의 테이블을 생성합니다. 공통 속성은 기본 테이블에, 타입별 속성은 각각의 테이블에 저장됩니다.

### 상세 설명

#### 기본 구성

```csharp
public abstract class Payment
{
    public int Id { get; set; }
    public decimal Amount { get; set; }
    public DateTime PaymentDate { get; set; }
}

public class CreditCardPayment : Payment
{
    public string CardNumber { get; set; } = string.Empty;
}

public class BankTransferPayment : Payment
{
    public string AccountNumber { get; set; } = string.Empty;
}

// Fluent API
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Payment>().ToTable("Payments");
    modelBuilder.Entity<CreditCardPayment>().ToTable("CreditCardPayments");
    modelBuilder.Entity<BankTransferPayment>().ToTable("BankTransferPayments");
}
```

#### 생성되는 테이블

```sql
-- 기본 테이블
CREATE TABLE Payments (
    Id int NOT NULL IDENTITY PRIMARY KEY,
    Amount decimal(18,2) NOT NULL,
    PaymentDate datetime2 NOT NULL
);

-- CreditCard 전용 테이블
CREATE TABLE CreditCardPayments (
    Id int NOT NULL PRIMARY KEY,
    CardNumber nvarchar(max) NOT NULL,
    CONSTRAINT FK_CreditCardPayments_Payments_Id
        FOREIGN KEY (Id) REFERENCES Payments(Id) ON DELETE CASCADE
);

-- BankTransfer 전용 테이블
CREATE TABLE BankTransferPayments (
    Id int NOT NULL PRIMARY KEY,
    AccountNumber nvarchar(max) NOT NULL,
    CONSTRAINT FK_BankTransferPayments_Payments_Id
        FOREIGN KEY (Id) REFERENCES Payments(Id) ON DELETE CASCADE
);
```

#### 테이블 구조

```
┌──────────────────┐
│    Payments      │
├────┬───────┬─────┤
│ Id │Amount │Date │
├────┼───────┼─────┤
│ 1  │100.00 │...  │
│ 2  │200.00 │...  │
│ 3  │150.00 │...  │
└────┴───────┴─────┘
         │
    ┌────┴────┐
    ▼         ▼
┌─────────────────┐   ┌─────────────────────┐
│CreditCardPayments│  │BankTransferPayments │
├────┬────────────┤   ├────┬────────────────┤
│ Id │ CardNumber │   │ Id │ AccountNumber  │
├────┼────────────┤   ├────┼────────────────┤
│ 1  │ 4111...    │   │ 2  │ 110-2345-...   │
│ 3  │ 5500...    │   └────┴────────────────┘
└────┴────────────┘
```

#### 쿼리 동작

```csharp
// 특정 타입 조회
var creditCards = await context.Set<CreditCardPayment>().ToListAsync();

// 생성되는 SQL (JOIN 필요)
// SELECT p.Id, p.Amount, p.PaymentDate, c.CardNumber
// FROM Payments p
// INNER JOIN CreditCardPayments c ON p.Id = c.Id

// 전체 조회 시 (모든 타입)
var allPayments = await context.Payments.ToListAsync();

// 생성되는 SQL (LEFT JOIN 다수)
// SELECT p.Id, p.Amount, p.PaymentDate, c.CardNumber, b.AccountNumber
// FROM Payments p
// LEFT JOIN CreditCardPayments c ON p.Id = c.Id
// LEFT JOIN BankTransferPayments b ON p.Id = b.Id
```

### TPT 장단점

| 장점 | 단점 |
|------|------|
| 정규화된 스키마 | JOIN으로 성능 저하 |
| NULL 컬럼 없음 | 복잡한 쿼리 생성 |
| 각 테이블에 제약 가능 | 삽입/업데이트 복잡 |
| 타입별 테이블 관리 용이 | 전체 조회 시 느림 |

---

## 6.3 TPC (Table Per Concrete Class)

### 간략 설명
각 구체 클래스마다 모든 속성을 포함한 독립적인 테이블을 생성합니다. EF Core 7+에서 지원됩니다.

### 상세 설명

#### 기본 구성

```csharp
public abstract class Payment
{
    public int Id { get; set; }
    public decimal Amount { get; set; }
    public DateTime PaymentDate { get; set; }
}

public class CreditCardPayment : Payment
{
    public string CardNumber { get; set; } = string.Empty;
}

public class BankTransferPayment : Payment
{
    public string AccountNumber { get; set; } = string.Empty;
}

// Fluent API (EF Core 7+)
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Payment>().UseTpcMappingStrategy();

    modelBuilder.Entity<CreditCardPayment>().ToTable("CreditCardPayments");
    modelBuilder.Entity<BankTransferPayment>().ToTable("BankTransferPayments");
}
```

#### 생성되는 테이블

```sql
-- 시퀀스 (ID 고유성 보장)
CREATE SEQUENCE PaymentIds AS int START WITH 1;

-- CreditCard 전용 테이블 (모든 속성 포함)
CREATE TABLE CreditCardPayments (
    Id int NOT NULL DEFAULT NEXT VALUE FOR PaymentIds PRIMARY KEY,
    Amount decimal(18,2) NOT NULL,
    PaymentDate datetime2 NOT NULL,
    CardNumber nvarchar(max) NOT NULL
);

-- BankTransfer 전용 테이블 (모든 속성 포함)
CREATE TABLE BankTransferPayments (
    Id int NOT NULL DEFAULT NEXT VALUE FOR PaymentIds PRIMARY KEY,
    Amount decimal(18,2) NOT NULL,
    PaymentDate datetime2 NOT NULL,
    AccountNumber nvarchar(max) NOT NULL
);
```

#### 테이블 구조

```
┌────────────────────────────────────────┐
│         CreditCardPayments             │
├────┬────────┬────────────┬─────────────┤
│ Id │ Amount │ PaymentDate│ CardNumber  │
├────┼────────┼────────────┼─────────────┤
│ 1  │ 100.00 │ 2024-01-01 │ 4111...     │
│ 3  │ 150.00 │ 2024-01-03 │ 5500...     │
└────┴────────┴────────────┴─────────────┘

┌────────────────────────────────────────┐
│        BankTransferPayments            │
├────┬────────┬────────────┬─────────────┤
│ Id │ Amount │ PaymentDate│AccountNumber│
├────┼────────┼────────────┼─────────────┤
│ 2  │ 200.00 │ 2024-01-02 │ 110-234...  │
└────┴────────┴────────────┴─────────────┘
```

#### 쿼리 동작

```csharp
// 특정 타입 조회 (단일 테이블)
var creditCards = await context.Set<CreditCardPayment>().ToListAsync();
// SELECT * FROM CreditCardPayments

// 전체 조회 (UNION ALL)
var allPayments = await context.Payments.ToListAsync();

// 생성되는 SQL
// SELECT Id, Amount, PaymentDate, CardNumber, NULL AS AccountNumber
// FROM CreditCardPayments
// UNION ALL
// SELECT Id, Amount, PaymentDate, NULL AS CardNumber, AccountNumber
// FROM BankTransferPayments
```

### TPC 장단점

| 장점 | 단점 |
|------|------|
| 개별 조회 시 JOIN 없음 | 전체 조회 시 UNION 필요 |
| 독립적인 테이블 구조 | ID 시퀀스 관리 필요 |
| 타입별 인덱싱 최적화 | 공통 속성 중복 |
| 파티셔닝 용이 | 스키마 변경 시 복잡 |

---

## 6.4 전략별 장단점 비교

### 비교 표

| 기준 | TPH | TPT | TPC |
|------|-----|-----|-----|
| 테이블 수 | 1개 | N+1개 | N개 |
| NULL 컬럼 | 많음 | 없음 | 없음 |
| 정규화 | 낮음 | 높음 | 중간 |
| 단일 타입 조회 | 빠름 | 느림 (JOIN) | 빠름 |
| 전체 조회 | 매우 빠름 | 느림 | 중간 (UNION) |
| 삽입 성능 | 빠름 | 느림 | 빠름 |
| 제약 조건 | 어려움 | 쉬움 | 쉬움 |
| 스키마 변경 | 쉬움 | 중간 | 어려움 |
| EF Core 버전 | 모든 버전 | 모든 버전 | 7.0+ |

### 선택 가이드

```
상속 매핑 전략 선택 플로우:

시작
 │
 ├─ 타입 수가 적고 속성 차이가 적은가?
 │   └─ Yes → TPH (가장 단순하고 빠름)
 │
 ├─ 타입별로 NULL 불가 제약이 필요한가?
 │   └─ Yes → TPT 또는 TPC
 │
 ├─ 전체 계층을 자주 조회하는가?
 │   ├─ Yes → TPH
 │   └─ No  → 특정 타입만 조회한다면 TPC
 │
 ├─ 타입별로 다른 인덱스/파티셔닝이 필요한가?
 │   └─ Yes → TPC
 │
 └─ 기존 데이터베이스 스키마가 이미 존재하는가?
     └─ 스키마에 맞는 전략 선택
```

### 실제 시나리오별 권장

```csharp
// 시나리오 1: 알림 시스템 - TPH 권장
// 이유: 모든 알림을 함께 조회하는 경우가 많음
public abstract class Notification
{
    public int Id { get; set; }
    public string Message { get; set; } = string.Empty;
    public DateTime CreatedAt { get; set; }
    public bool IsRead { get; set; }
}

public class EmailNotification : Notification
{
    public string EmailAddress { get; set; } = string.Empty;
}

public class PushNotification : Notification
{
    public string DeviceToken { get; set; } = string.Empty;
}

// 시나리오 2: 결제 시스템 - TPC 권장
// 이유: 결제 유형별 규제/감사 요구사항이 다름
public abstract class Payment { ... }
public class CreditCardPayment : Payment { ... }  // PCI-DSS 준수
public class BankTransferPayment : Payment { ... }  // 은행 규제

// 시나리오 3: 전자상거래 상품 - TPT 권장
// 이유: 카테고리별 속성이 매우 다르고 제약 조건 필요
public abstract class Product { ... }
public class Book : Product
{
    public string ISBN { get; set; }  // NOT NULL, UNIQUE
    public string Author { get; set; }
}
public class Electronics : Product
{
    public string Voltage { get; set; }  // 필수
    public int WarrantyMonths { get; set; }
}
```

### 혼합 사용

```csharp
// 복잡한 계층에서 혼합 가능
public abstract class Content
{
    public int Id { get; set; }
    public string Title { get; set; } = string.Empty;
}

// 미디어는 TPC (대용량, 독립적)
public abstract class Media : Content
{
    public string Url { get; set; } = string.Empty;
}

public class Video : Media
{
    public int DurationSeconds { get; set; }
}

public class Audio : Media
{
    public int Bitrate { get; set; }
}

// 문서는 TPH (비슷한 구조)
public abstract class Document : Content
{
    public string Body { get; set; } = string.Empty;
}

public class Article : Document
{
    public string Author { get; set; } = string.Empty;
}

public class Tutorial : Document
{
    public string[] Tags { get; set; } = Array.Empty<string>();
}

protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    // Media 계층은 TPC
    modelBuilder.Entity<Media>().UseTpcMappingStrategy();

    // Document 계층은 TPH
    modelBuilder.Entity<Document>().UseTphMappingStrategy();
}
```

---

## 요약

| 전략 | 사용 시나리오 | 핵심 특징 |
|------|-------------|----------|
| TPH | 타입 수 적음, 전체 조회 많음 | 단일 테이블, 구분자 컬럼 |
| TPT | 정규화 필요, 타입별 제약 | 타입당 테이블, JOIN |
| TPC | 타입별 독립성, 대용량 | 완전 분리, UNION |

## 다음 장 예고

다음 장에서는 값 객체와 소유 타입을 사용하여 복잡한 도메인 모델을 구성하는 방법을 알아봅니다.
