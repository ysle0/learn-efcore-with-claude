# Chapter 09: 값 객체와 소유 타입

## 개요

값 객체(Value Object)와 소유 타입(Owned Type)을 사용하여 풍부한 도메인 모델을 구성하는 방법을 알아봅니다. 값 변환기, 소유 타입, 그리고 EF Core 8+의 복합 타입까지 다룹니다.

---

## 7.1 값 변환기(Value Converters)

### 간략 설명
값 변환기는 엔티티 속성 값을 데이터베이스에 저장할 때와 읽을 때 변환하는 기능입니다. 커스텀 타입을 데이터베이스 기본 타입으로 매핑할 수 있습니다.

### 기본 사용법

```csharp
// 열거형을 문자열로 저장
public class Order
{
    public int Id { get; set; }
    public OrderStatus Status { get; set; }
}

public enum OrderStatus
{
    Pending,
    Confirmed,
    Shipped,
    Delivered,
    Cancelled
}

// Fluent API
modelBuilder.Entity<Order>()
    .Property(o => o.Status)
    .HasConversion<string>();  // 문자열로 저장

// 또는 명시적 변환
modelBuilder.Entity<Order>()
    .Property(o => o.Status)
    .HasConversion(
        v => v.ToString(),                          // C# → DB
        v => Enum.Parse<OrderStatus>(v));           // DB → C#
```

### 내장 변환기

```csharp
using Microsoft.EntityFrameworkCore.Storage.ValueConversion;

// 1. BoolToStringConverter
modelBuilder.Entity<User>()
    .Property(u => u.IsActive)
    .HasConversion(new BoolToStringConverter("Inactive", "Active"));

// 2. BoolToZeroOneConverter
modelBuilder.Entity<User>()
    .Property(u => u.IsVerified)
    .HasConversion<BoolToZeroOneConverter<int>>();

// 3. EnumToStringConverter
modelBuilder.Entity<Order>()
    .Property(o => o.Status)
    .HasConversion(new EnumToStringConverter<OrderStatus>());

// 4. DateTimeOffsetToBinaryConverter
modelBuilder.Entity<Event>()
    .Property(e => e.StartTime)
    .HasConversion<DateTimeOffsetToBinaryConverter>();

// 5. TimeSpanToTicksConverter
modelBuilder.Entity<Video>()
    .Property(v => v.Duration)
    .HasConversion<TimeSpanToTicksConverter>();
```

### 커스텀 값 변환기

```csharp
// 도메인 객체를 위한 값 변환기
public class Money
{
    public decimal Amount { get; }
    public string Currency { get; }

    public Money(decimal amount, string currency)
    {
        Amount = amount;
        Currency = currency;
    }

    public override string ToString() => $"{Amount:F2} {Currency}";
}

// 변환기 클래스
public class MoneyToStringConverter : ValueConverter<Money, string>
{
    public MoneyToStringConverter()
        : base(
            money => $"{money.Amount}|{money.Currency}",           // C# → DB
            str => ParseMoney(str))                                 // DB → C#
    {
    }

    private static Money ParseMoney(string value)
    {
        var parts = value.Split('|');
        return new Money(decimal.Parse(parts[0]), parts[1]);
    }
}

// 적용
modelBuilder.Entity<Product>()
    .Property(p => p.Price)
    .HasConversion(new MoneyToStringConverter());
```

### 복잡한 변환 예제

```csharp
// JSON으로 저장
public class UserPreferences
{
    public string Theme { get; set; } = "light";
    public string Language { get; set; } = "ko";
    public bool EmailNotifications { get; set; } = true;
}

public class JsonConverter<T> : ValueConverter<T, string> where T : class
{
    public JsonConverter()
        : base(
            v => JsonSerializer.Serialize(v, (JsonSerializerOptions?)null),
            v => JsonSerializer.Deserialize<T>(v, (JsonSerializerOptions?)null)!)
    {
    }
}

modelBuilder.Entity<User>()
    .Property(u => u.Preferences)
    .HasConversion(new JsonConverter<UserPreferences>())
    .HasColumnType("nvarchar(max)");

// 암호화 변환기
public class EncryptedConverter : ValueConverter<string, string>
{
    public EncryptedConverter(IEncryptionService encryption)
        : base(
            v => encryption.Encrypt(v),
            v => encryption.Decrypt(v))
    {
    }
}
```

### 값 비교자 (Value Comparer)

```csharp
// 컬렉션이나 복잡한 타입은 비교자도 필요
public class StringListConverter : ValueConverter<List<string>, string>
{
    public StringListConverter()
        : base(
            v => string.Join(",", v),
            v => v.Split(',', StringSplitOptions.RemoveEmptyEntries).ToList())
    {
    }
}

modelBuilder.Entity<Post>()
    .Property(p => p.Tags)
    .HasConversion(new StringListConverter())
    .Metadata.SetValueComparer(new ValueComparer<List<string>>(
        (c1, c2) => c1!.SequenceEqual(c2!),  // 같음 비교
        c => c.Aggregate(0, (a, v) => HashCode.Combine(a, v.GetHashCode())),  // 해시
        c => c.ToList()));  // 스냅샷
```

---

## 7.2 소유 타입(Owned Types)

### 간략 설명
소유 타입은 다른 엔티티 타입의 일부로만 존재하는 타입입니다. 자체 테이블이 없고 소유자 엔티티의 테이블에 포함되거나 별도 테이블로 분리될 수 있습니다.

### 기본 사용법

```csharp
// 값 객체 정의
public class Address
{
    public string Street { get; set; } = string.Empty;
    public string City { get; set; } = string.Empty;
    public string ZipCode { get; set; } = string.Empty;
    public string Country { get; set; } = string.Empty;
}

// 엔티티에서 사용
public class Customer
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;

    // 소유 타입
    public Address ShippingAddress { get; set; } = null!;
    public Address? BillingAddress { get; set; }
}

// Fluent API
modelBuilder.Entity<Customer>(entity =>
{
    entity.OwnsOne(c => c.ShippingAddress, address =>
    {
        address.Property(a => a.Street).HasColumnName("ShippingStreet");
        address.Property(a => a.City).HasColumnName("ShippingCity");
        address.Property(a => a.ZipCode).HasColumnName("ShippingZipCode");
        address.Property(a => a.Country).HasColumnName("ShippingCountry");
    });

    entity.OwnsOne(c => c.BillingAddress, address =>
    {
        address.Property(a => a.Street).HasColumnName("BillingStreet");
        address.Property(a => a.City).HasColumnName("BillingCity");
        address.Property(a => a.ZipCode).HasColumnName("BillingZipCode");
        address.Property(a => a.Country).HasColumnName("BillingCountry");
    });
});
```

### 생성되는 테이블

```sql
CREATE TABLE Customers (
    Id int NOT NULL IDENTITY PRIMARY KEY,
    Name nvarchar(max) NOT NULL,

    -- ShippingAddress (필수)
    ShippingStreet nvarchar(max) NOT NULL,
    ShippingCity nvarchar(max) NOT NULL,
    ShippingZipCode nvarchar(max) NOT NULL,
    ShippingCountry nvarchar(max) NOT NULL,

    -- BillingAddress (선택적)
    BillingStreet nvarchar(max) NULL,
    BillingCity nvarchar(max) NULL,
    BillingZipCode nvarchar(max) NULL,
    BillingCountry nvarchar(max) NULL
);
```

### 별도 테이블로 분리

```csharp
modelBuilder.Entity<Customer>()
    .OwnsOne(c => c.ShippingAddress, address =>
    {
        address.ToTable("CustomerShippingAddresses");
    });
```

### 소유 컬렉션

```csharp
// 주문에 여러 주문 항목 (소유 타입)
public class Order
{
    public int Id { get; set; }
    public string OrderNumber { get; set; } = string.Empty;

    public ICollection<OrderItem> Items { get; set; } = new List<OrderItem>();
}

public class OrderItem
{
    public string ProductName { get; set; } = string.Empty;
    public int Quantity { get; set; }
    public decimal UnitPrice { get; set; }
}

// Fluent API
modelBuilder.Entity<Order>()
    .OwnsMany(o => o.Items, item =>
    {
        item.ToTable("OrderItems");
        item.WithOwner().HasForeignKey("OrderId");
        item.Property<int>("Id");
        item.HasKey("Id");
    });
```

### 중첩된 소유 타입

```csharp
public class Customer
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public ContactInfo ContactInfo { get; set; } = null!;
}

public class ContactInfo
{
    public string Email { get; set; } = string.Empty;
    public string Phone { get; set; } = string.Empty;
    public Address Address { get; set; } = null!;  // 중첩된 소유 타입
}

public class Address
{
    public string Street { get; set; } = string.Empty;
    public string City { get; set; } = string.Empty;
}

modelBuilder.Entity<Customer>()
    .OwnsOne(c => c.ContactInfo, contact =>
    {
        contact.Property(ci => ci.Email).HasColumnName("Email");
        contact.Property(ci => ci.Phone).HasColumnName("Phone");

        contact.OwnsOne(ci => ci.Address, address =>
        {
            address.Property(a => a.Street).HasColumnName("Street");
            address.Property(a => a.City).HasColumnName("City");
        });
    });
```

---

## 7.3 복합 타입(Complex Types) - EF Core 8+

### 간략 설명
EF Core 8에서 도입된 복합 타입은 소유 타입의 간소화 버전입니다. 값 객체를 더 쉽게 매핑할 수 있습니다.

### 기본 사용법

```csharp
// 복합 타입 정의
[ComplexType]  // 또는 Fluent API 사용
public class Address
{
    public required string Street { get; set; }
    public required string City { get; set; }
    public required string ZipCode { get; set; }
}

public class Customer
{
    public int Id { get; set; }
    public required string Name { get; set; }
    public required Address ShippingAddress { get; set; }
}

// Fluent API
modelBuilder.Entity<Customer>()
    .ComplexProperty(c => c.ShippingAddress);
```

### 소유 타입 vs 복합 타입

| 특성 | 소유 타입 (Owned) | 복합 타입 (Complex) |
|------|------------------|-------------------|
| Nullable | 가능 | 불가능 (항상 필수) |
| 별도 테이블 | 가능 | 불가능 |
| 컬렉션 | 가능 (OwnsMany) | 불가능 |
| 중첩 | 가능 | 가능 |
| 설정 | 더 유연함 | 더 단순함 |
| 성능 | 동일 | 동일 |

### 복합 타입 구성

```csharp
modelBuilder.Entity<Customer>(entity =>
{
    entity.ComplexProperty(c => c.ShippingAddress, address =>
    {
        address.Property(a => a.Street)
            .HasMaxLength(200)
            .HasColumnName("ShippingStreet");

        address.Property(a => a.City)
            .HasMaxLength(100)
            .HasColumnName("ShippingCity");

        address.Property(a => a.ZipCode)
            .HasMaxLength(10)
            .HasColumnName("ShippingZipCode");
    });
});
```

---

## 도메인 주도 설계와 값 객체

### 진정한 값 객체 구현

```csharp
// 불변 값 객체
public sealed class Money : IEquatable<Money>
{
    public decimal Amount { get; }
    public Currency Currency { get; }

    public Money(decimal amount, Currency currency)
    {
        if (amount < 0)
            throw new ArgumentException("금액은 0 이상이어야 합니다.", nameof(amount));

        Amount = amount;
        Currency = currency;
    }

    // 팩토리 메서드
    public static Money FromKRW(decimal amount) => new(amount, Currency.KRW);
    public static Money FromUSD(decimal amount) => new(amount, Currency.USD);
    public static Money Zero(Currency currency) => new(0, currency);

    // 연산자 오버로딩
    public static Money operator +(Money left, Money right)
    {
        if (left.Currency != right.Currency)
            throw new InvalidOperationException("통화가 다릅니다.");
        return new Money(left.Amount + right.Amount, left.Currency);
    }

    public static Money operator *(Money money, decimal multiplier)
        => new(money.Amount * multiplier, money.Currency);

    // 값 동등성
    public bool Equals(Money? other)
    {
        if (other is null) return false;
        return Amount == other.Amount && Currency == other.Currency;
    }

    public override bool Equals(object? obj) => Equals(obj as Money);

    public override int GetHashCode() => HashCode.Combine(Amount, Currency);

    public static bool operator ==(Money? left, Money? right)
        => EqualityComparer<Money>.Default.Equals(left, right);

    public static bool operator !=(Money? left, Money? right)
        => !(left == right);

    public override string ToString() => $"{Amount:N2} {Currency}";
}

public enum Currency { KRW, USD, EUR, JPY }
```

### EF Core 매핑

```csharp
// 값 변환기 방식
public class MoneyConverter : ValueConverter<Money, decimal>
{
    public MoneyConverter()
        : base(
            m => m.Amount,
            a => Money.FromKRW(a))  // 단일 통화 가정
    {
    }
}

// 또는 소유 타입 방식
modelBuilder.Entity<Product>()
    .OwnsOne(p => p.Price, price =>
    {
        price.Property(m => m.Amount).HasColumnName("Price");
        price.Property(m => m.Currency).HasColumnName("PriceCurrency");
    });
```

### 값 객체 컬렉션

```csharp
// 태그 목록 값 객체
public sealed class TagCollection : IReadOnlyCollection<string>
{
    private readonly HashSet<string> _tags;

    public TagCollection() : this(Array.Empty<string>()) { }

    public TagCollection(IEnumerable<string> tags)
    {
        _tags = new HashSet<string>(
            tags.Select(t => t.ToLowerInvariant().Trim())
                .Where(t => !string.IsNullOrEmpty(t)));
    }

    public int Count => _tags.Count;

    public bool Contains(string tag) => _tags.Contains(tag.ToLowerInvariant());

    public TagCollection Add(string tag)
        => new(_tags.Append(tag));

    public TagCollection Remove(string tag)
        => new(_tags.Where(t => t != tag.ToLowerInvariant()));

    public IEnumerator<string> GetEnumerator() => _tags.GetEnumerator();
    IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();

    public override string ToString() => string.Join(", ", _tags);
}

// 변환기
public class TagCollectionConverter : ValueConverter<TagCollection, string>
{
    public TagCollectionConverter()
        : base(
            tc => string.Join(",", tc),
            s => new TagCollection(s.Split(',', StringSplitOptions.RemoveEmptyEntries)))
    {
    }
}

modelBuilder.Entity<Post>()
    .Property(p => p.Tags)
    .HasConversion(new TagCollectionConverter())
    .Metadata.SetValueComparer(new ValueComparer<TagCollection>(
        (c1, c2) => c1!.SequenceEqual(c2!),
        c => c.Aggregate(0, (a, v) => HashCode.Combine(a, v.GetHashCode())),
        c => new TagCollection(c)));
```

---

## JSON 컬럼 (EF Core 7+)

### 기본 사용법

```csharp
public class Order
{
    public int Id { get; set; }
    public string OrderNumber { get; set; } = string.Empty;

    // JSON으로 저장될 속성
    public ShippingInfo Shipping { get; set; } = null!;
}

public class ShippingInfo
{
    public string Carrier { get; set; } = string.Empty;
    public string TrackingNumber { get; set; } = string.Empty;
    public Address DeliveryAddress { get; set; } = null!;
}

public class Address
{
    public string Street { get; set; } = string.Empty;
    public string City { get; set; } = string.Empty;
}

// Fluent API
modelBuilder.Entity<Order>()
    .OwnsOne(o => o.Shipping, shipping =>
    {
        shipping.ToJson();  // JSON 컬럼으로 저장

        shipping.OwnsOne(s => s.DeliveryAddress);
    });
```

### 생성되는 테이블

```sql
CREATE TABLE Orders (
    Id int NOT NULL IDENTITY PRIMARY KEY,
    OrderNumber nvarchar(max) NOT NULL,
    Shipping nvarchar(max) NOT NULL  -- JSON 데이터
);

-- 저장되는 JSON 예시
-- {
--   "Carrier": "FedEx",
--   "TrackingNumber": "123456789",
--   "DeliveryAddress": {
--     "Street": "123 Main St",
--     "City": "Seoul"
--   }
-- }
```

### JSON 쿼리

```csharp
// JSON 내부 속성 쿼리 가능
var orders = await context.Orders
    .Where(o => o.Shipping.Carrier == "FedEx")
    .ToListAsync();

var seoulOrders = await context.Orders
    .Where(o => o.Shipping.DeliveryAddress.City == "Seoul")
    .ToListAsync();

// 생성되는 SQL (SQL Server)
// SELECT * FROM Orders WHERE JSON_VALUE(Shipping, '$.Carrier') = 'FedEx'
```

### JSON 컬렉션

```csharp
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;

    public List<ProductAttribute> Attributes { get; set; } = new();
}

public class ProductAttribute
{
    public string Name { get; set; } = string.Empty;
    public string Value { get; set; } = string.Empty;
}

modelBuilder.Entity<Product>()
    .OwnsMany(p => p.Attributes, attr =>
    {
        attr.ToJson();
    });

// 쿼리
var products = await context.Products
    .Where(p => p.Attributes.Any(a => a.Name == "Color" && a.Value == "Red"))
    .ToListAsync();
```

---

## 요약

| 기능 | 사용 시나리오 | EF Core 버전 |
|------|-------------|-------------|
| Value Converter | 단순 타입 변환 | 모든 버전 |
| Owned Type | 복잡한 값 객체, nullable 지원 | 2.0+ |
| Complex Type | 간단한 값 객체, 필수 속성 | 8.0+ |
| JSON Column | 유연한 구조, 중첩 데이터 | 7.0+ |

## 다음 장 예고

다음 Part에서는 LINQ 쿼리 기초부터 고급 쿼리 기법까지 데이터 조회 방법을 상세히 알아봅니다.
