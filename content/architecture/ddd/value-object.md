---
title: Value Object — 값으로 다루는 객체
type: concept
tags: [ddd, value-object, domain-modeling, immutability, java]
related:
  - "[[../../language/java/record]]"
last_reviewed: 2026-05-03
publish: false
---

# Value Object — 값으로 다루는 객체

## 목차

- [한 줄 정의](#한-줄-정의)
- [Entity vs Value Object](#entity-vs-value-object)
- [VO의 5가지 핵심 용도](#vo의-5가지-핵심-용도)
- [어떤 걸 VO로 만들까](#어떤-걸-vo로-만들까)
- [VO vs DTO](#vo-vs-dto)
- [흔히 하는 실수](#흔히-하는-실수)
- [Java에서 VO 구현 — record가 사실상 표준](#java에서-vo-구현--record가-사실상-표준)
- [핵심 정리](#핵심-정리)

## 한 줄 정의

**값(value) 그 자체가 정체성인 객체.** ID로 구분되는 게 아니라, **들고 있는 속성들이 같으면 같은 것**으로 취급되는 객체.

```java
Money a = new Money(1000, KRW);
Money b = new Money(1000, KRW);

a.equals(b);   // true — 값이 같으니 같은 것
```

`Money 1000원`은 누가 만들었든, 언제 만들었든, 어디에 있든 — **같은 1000원**.

## Entity vs Value Object

DDD(Domain-Driven Design)에서 객체는 크게 두 부류로 나뉜다.

### Entity — ID로 구분

```java
class User {
    Long id;       // ← 이게 정체성
    String name;
    String email;
}

User u1 = new User(1L, "Alice", "alice@example.com");
User u2 = new User(1L, "Alice (renamed)", "alice@example.com");
// id가 같으면 같은 사람. 이름이 바뀌어도 동일한 User.
```

User는 **이름이 바뀌어도, 이메일이 바뀌어도 같은 사용자**. id가 그를 정의한다.

### Value Object — 값으로 구분

```java
record Money(BigDecimal amount, Currency currency) { }

Money m1 = new Money(1000, KRW);
Money m2 = new Money(1000, KRW);
Money m3 = new Money(2000, KRW);
// m1 == m2 (값 같음 → 같음), m1 ≠ m3 (값 다름 → 다름)
```

Money는 **값이 바뀌면 다른 객체**. "1000원"과 "2000원"은 그냥 다른 두 값.

### 비유

- **Entity**: 사람. 머리 자르고 살 빠지고 옷 갈아입어도 그 사람. 주민번호로 식별.
- **Value Object**: 돈. 1000원짜리 두 장은 어느 게 어느 건지 구분할 필요 없음. 값으로 식별.

### 비교표

| | Entity | Value Object |
|---|---|---|
| 정체성 | ID (식별자) | 속성 값 |
| 동등성 비교 | ID가 같으면 같음 | 모든 속성이 같으면 같음 |
| 가변성 | 가변 (시간에 따라 변함) | 불변이 자연스러움 |
| 라이프사이클 | 생성 → 변경 → 삭제 | 생성 → 폐기 (변경 없음) |
| 영속화 | 자체 테이블/문서 | 보통 Entity 내 임베드 |
| 예시 | `User`, `Order`, `Product` | `Money`, `Email`, `Address`, `DateRange` |

## VO의 5가지 핵심 용도

### 1. 타입으로 의미 강제 (Primitive obsession 방지)

> **Primitive obsession** — 도메인 개념을 String/UUID/Long 같은 범용 타입으로만 표현하는 안티패턴. 타입이 너무 넓어 컴파일러가 의미적 실수를 잡지 못한다.

같은 `Long` 타입이면 그게 출금 계좌 ID인지 입금 계좌 ID인지 컴파일러는 구분할 수 없다. 인자 순서 실수 같은 버그가 운영에 나가서야 발견된다.

```java
// ❌ Primitive 그대로
void transfer(Long fromAccountId, Long toAccountId, Long amount) { }
transfer(toId, fromId, amount);   // 컴파일러가 못 잡음 — 운영에서 폭발

// ✅ VO로 좁히기
void transfer(SourceAccount from, TargetAccount to, Money amount) { }
transfer(toAccount, fromAccount, amount);   // ❌ 컴파일 에러
```

도메인 개념마다 별도 타입으로 좁히면 같은 실수가 **컴파일 시점에** 잡힌다.

### 2. 검증을 한 곳에 모음 (Self-validating)

값이 유효한지 매번 검증하는 대신, **만들어지는 순간** 한 번만 검증한다. VO 인스턴스가 존재한다는 사실 자체가 "유효한 값"의 증명.

```java
// ❌ 검증이 호출부마다 흩어짐
class UserService {
    void register(String email) {
        if (!email.contains("@")) throw ...;
        if (email.length() > 100) throw ...;
    }
}
class NotificationService {
    void notify(String email) {
        if (!email.contains("@")) throw ...;   // 또 검증
    }
}

// ✅ VO에서 한 번
public record Email(String value) {
    public Email {
        Objects.requireNonNull(value);
        if (!value.contains("@")) throw new IllegalArgumentException();
        if (value.length() > 100) throw new IllegalArgumentException();
        value = value.trim().toLowerCase();
    }
}

// 사용처에선 검증 코드 0줄
class UserService {
    void register(Email email) { ... }   // 이미 유효함이 보장됨
}
class NotificationService {
    void notify(Email email) { ... }     // 여기도 마찬가지
}
```

**"잘못된 상태의 객체가 존재할 수 없다(parse, don't validate)"** — 함수형 설계 원칙의 실천.

### 3. 도메인 행위의 자연스러운 위치

값과 관련된 로직을 그 값의 타입에 두면, 코드가 도메인 언어로 읽힌다.

#### 시나리오 — 주문 합계 계산

요구사항:
- 주문(Order)에는 여러 상품(LineItem)이 들어있고, 각 상품마다 가격(price)이 있다.
- 모든 상품 가격을 합한다.
- 합계가 임계값(`THRESHOLD`)을 넘으면 할인율(`DISCOUNT_RATE`)을 곱해 할인된 금액을 돌려준다.

#### ❌ 절차적 접근 — 가격을 그냥 BigDecimal로

가격을 단순 `BigDecimal`로 들고 다니면, 합계 계산 로직이 서비스 레이어로 빠져나간다.

```java
class LineItem {
    BigDecimal price;          // 가격이 그냥 BigDecimal
    // ...
}

class Order {
    List<LineItem> items;
    // 합계 계산 로직 없음 — 어디 있어야 할지 애매함
}

class OrderService {
    // 가격 리스트를 받아서 합계+할인을 직접 계산
    BigDecimal calculateTotal(List<BigDecimal> prices) {
        BigDecimal sum = BigDecimal.ZERO;
        for (BigDecimal p : prices) {
            sum = sum.add(p);
        }
        if (sum.compareTo(THRESHOLD) > 0) {
            sum = sum.multiply(DISCOUNT_RATE);
        }
        return sum;
    }
}

// 사용처
List<BigDecimal> prices = order.items.stream()
    .map(item -> item.price)
    .toList();
BigDecimal total = orderService.calculateTotal(prices);
```

문제점:
- "합계가 임계값을 넘으면 할인" 로직이 **`OrderService` 안의 BigDecimal 계산**으로 표현됨 → 도메인 의미가 산술 코드에 묻힘
- `BigDecimal`엔 통화(KRW/USD) 정보가 없음 → 다른 통화를 섞어도 컴파일러가 못 잡음
- 다른 서비스에서 합계가 또 필요하면? 같은 로직을 또 구현 (중복)

#### ✅ VO 기반 접근 — Money 타입에 도메인 행위 응집

가격을 `Money` VO로 격상하고, 도메인 행위(`add`, `multiply`, `isGreaterThan`)를 그 안에 둔다.

```java
// 1. 값과 행위를 같이 가진 VO
public record Money(BigDecimal amount, Currency currency) {
    public static final Money ZERO_KRW = new Money(BigDecimal.ZERO, KRW);

    public Money add(Money other) {
        // 통화가 다르면 더할 수 없음 (도메인 규칙도 VO가 강제)
        if (!currency.equals(other.currency))
            throw new IllegalArgumentException("currency mismatch");
        return new Money(amount.add(other.amount), currency);
    }

    public Money multiply(BigDecimal factor) {
        return new Money(amount.multiply(factor), currency);
    }

    public boolean isGreaterThan(Money other) {
        return amount.compareTo(other.amount) > 0;
    }
}

// 2. LineItem은 가격을 Money로 보유
public record LineItem(ProductId productId, int quantity, Money price) { }

// 3. Order는 자기 items에 대한 합계를 자기가 안다
public class Order {
    private final List<LineItem> items;   // ← 이 items가 Order의 필드
    private final Money threshold;
    private final BigDecimal discountRate;

    public Money total() {
        // items의 가격들을 Money.add로 합산
        Money sum = items.stream()
            .map(LineItem::price)                       // 각 LineItem의 price (Money)
            .reduce(Money.ZERO_KRW, Money::add);        // Money끼리 합산

        // 임계값 초과면 할인 적용
        return sum.isGreaterThan(threshold)
            ? sum.multiply(discountRate)
            : sum;
    }
}

// 사용처
Money total = order.total();
```

#### 무엇이 좋아졌나

| | 절차적 (`OrderService`) | VO 기반 (`Order` + `Money`) |
|---|---|---|
| 합산 로직 위치 | OrderService 안의 BigDecimal for-loop | `Money.add` (값의 타입에) |
| 할인 로직 위치 | OrderService 안의 if + multiply | `Order.total()` (도메인 객체에) |
| 통화 안전성 | 없음 (KRW/USD 섞여도 통과) | `Money.add`가 통화 검증 |
| 재사용성 | OrderService 메서드 호출 필요 | `order.total()` 한 줄 |
| 가독성 | 산술 연산 나열 | `sum.isGreaterThan(threshold)` ← 도메인 언어 |

#### 도메인 언어로 읽힌다는 것

VO 버전의 `Order.total()` 본문을 한국어로 읽으면:

> "items의 가격들을 0원에서부터 더해서 합계를 만든다.
> 합계가 임계값보다 크면 할인율을 곱한 값을, 아니면 합계 그대로 돌려준다."

비즈니스 담당자가 요구사항 말할 때 쓰는 그대로의 문장. 반면 절차적 버전은 *"BigDecimal을 ZERO에서 시작해서 for로 add하고 compareTo가 0보다 크면 multiply한다"* — 이건 **개발자 언어**.

**값의 타입(Money)에 행위(add/multiply/isGreaterThan)를 붙이면, 그 값을 다루는 코드가 자연스럽게 도메인 언어가 된다.** 이게 VO가 "도메인 행위의 자연스러운 위치"인 이유.

### 4. 불변성으로 안전한 공유

여러 곳에서 동시에 참조해도 절대 안 바뀜 → 락 없이 공유 가능, 캐싱 자유, 부수효과 추적 단순.

```java
// 가변이면 — 어디서 바꿨는지 추적 지옥
class Address { String street; String city; }
Address a = ...;
service1.process(a);   // 안에서 a.street 바꿈?
service2.process(a);   // a가 바뀐 채로 들어옴

// 불변(VO)이면 — 누가 봐도 같은 값
record Address(String street, String city) { }
Address a = ...;
service1.process(a);   // a는 절대 안 바뀜
service2.process(a);   // 처음 그대로
```

스레드 안전성, 캐시 키 안전성, equals/hashCode 신뢰성 — 모든 게 불변에서 나온다.

### 5. Set/Map의 키, 컬렉션 비교에 적합

값 동등성이 자동으로 보장되니 컬렉션 키로 안전.

```java
record DateRange(LocalDate start, LocalDate end) { }

Map<DateRange, Discount> discounts = new HashMap<>();
discounts.put(new DateRange(d1, d2), discount);

// 다른 인스턴스로도 조회됨 — 값이 같으니까
discounts.get(new DateRange(d1, d2));   // ✅ discount 반환

Set<Email> uniqueEmails = ...;
uniqueEmails.add(new Email("a@example.com"));
uniqueEmails.add(new Email("a@example.com"));   // 중복 자동 제거
uniqueEmails.size();   // 1
```

## 어떤 걸 VO로 만들까

### 판별 체크리스트

| 질문 | Yes면 VO 후보 |
|---|---|
| 값이 같으면 같은 것으로 봐야 하나? | ✅ |
| 한 번 만들어지면 안 바뀌는 게 자연스러운가? | ✅ |
| 별도 ID 없이 속성만으로 정체성이 충분한가? | ✅ |
| 도메인에 의미 있는 개념인가? (단순 String/UUID가 아닌) | ✅ |
| 검증/계산 로직을 모을 자리가 필요한가? | ✅ |

### 전형적인 VO 후보

- **식별자**: `EventId`, `UserId`, `OrderId`
- **측정값**: `Money`, `Distance`, `Weight`, `Temperature`
- **좌표/범위**: `Point`, `DateRange`, `IpAddress`, `Coordinate`
- **도메인 문자열**: `Email`, `PhoneNumber`, `PostalCode`, `Url`
- **합성 값**: `FullName(first, last)`, `Address(street, city, zip)`

### VO로 만들면 안 되는 것

- 데이터베이스에서 ID로 추적되는 것 (`User`, `Order`, `Product`) → **Entity**
- 라이프사이클이 있는 것 (생성 → 변경 → 삭제) → **Entity**
- 식별자 자체가 본질인 것

## VO vs DTO

자주 혼동되지만 의도가 다르다.

| | DTO (Data Transfer Object) | Value Object |
|---|---|---|
| 목적 | 외부 경계에서 데이터 운반 | 도메인 개념 표현 |
| 행위 | 없음 (그냥 데이터 그릇) | **있음** (도메인 로직 포함) |
| 검증 | 보통 없음 (외부 입력 그대로) | 생성 시점에 강제 |
| 위치 | API 경계, 직렬화 레이어 | 도메인 모델 |
| 동등성 | 의미 없음 (그냥 운반용) | 핵심 속성 (값 동등성) |

```java
// DTO — 그냥 데이터 운반용
public record UserResponse(Long id, String name, String email) { }

// VO — 도메인 의미 + 행위
public record Email(String value) {
    public Email { /* 검증 */ }
    public String domain() { ... }
    public boolean isCorporate() { ... }
}
```

Java에서 record로 둘 다 만들 수 있지만 **의도가 다름**. DTO는 "껍데기", VO는 "도메인 시민".

## 흔히 하는 실수

### ❌ ID 필드만 있는 클래스를 Entity로 착각

```java
record EventId(UUID value) { }       // 이건 VO
class Event { EventId id; ... }      // 이건 Entity
```

`EventId` **자체는 값**이다 — UUID 두 개가 같으면 같은 EventId. 정체성이 ID인 건 `EventId`가 *식별하는 대상*인 `Event`. ID 클래스 자체와 ID로 식별되는 대상은 다른 층위.

### ❌ VO를 가변으로 만들기

```java
class Money {
    BigDecimal amount;
    public void add(Money other) {       // ❌ 자기 자신 변경
        this.amount = this.amount.add(other.amount);
    }
}
```

VO의 핵심 가치(안전한 공유, 동등성 신뢰)가 깨진다. 변경이 필요하면 **새 인스턴스 반환**:

```java
record Money(BigDecimal amount, Currency currency) {
    public Money add(Money other) {
        return new Money(amount.add(other.amount), currency);
    }
}
```

### ❌ VO에 영속성/외부 의존 끌고 들어오기

```java
record Email(String value) {
    public void sendNotification() { ... }   // ❌ 외부 시스템 호출
    public void saveToDatabase() { ... }     // ❌ 영속성
}
```

VO는 순수 도메인 값. **부수효과/외부 의존 메서드는 서비스/리포지토리에**.

## Java에서 VO 구현 — record가 사실상 표준

Java 16+에서 [[../../language/java/record|record]]는 VO의 기술적 요구사항을 자동으로 채워준다.

| VO가 요구하는 것 | record가 제공 |
|---|---|
| 불변성 | 모든 컴포넌트 자동 final |
| 값 동등성 | equals/hashCode 자동 생성 (필드 기반) |
| 검증 자리 | compact constructor |
| 도메인 행위 추가 | 메서드 정의 자유 |
| 식별 가능한 의도 | `record` 키워드 자체가 "값 객체" 선언 |

```java
public record Email(String value) {
    public Email {                                    // 검증
        Objects.requireNonNull(value);
        if (!value.contains("@"))
            throw new IllegalArgumentException();
        value = value.trim().toLowerCase();           // 정규화
    }

    public String domain() {                          // 도메인 행위
        return value.substring(value.indexOf('@') + 1);
    }
}
```

record 이전에는 Lombok `@Value`로 흉내냈지만, record는 **언어 차원의 표준**이라 더 신뢰할 수 있다 (라이브러리 의존 없음, JVM 레벨에서 record로 식별됨).

## 핵심 정리

> **VO는 "값으로 다뤄야 할 도메인 개념"을 명시적인 타입으로 격상시키는 도구.**

### 효과

1. 컴파일러가 **의미적 실수**를 잡아준다 (primitive obsession 해소)
2. **검증이 한 곳**에 모인다 (생성 시점에 강제)
3. **도메인 로직**이 자연스러운 자리를 찾는다 (값 + 행위 응집)
4. **불변성**이 동시성/캐싱/추론을 단순화한다
5. **컬렉션 키**로 안전하게 사용 가능

### 사고 전환

> **Primitive 사고**: "이 값은 String이다, UUID다, Long이다"
> **VO 사고**: "이 값은 Email이다, EventId다, Money다 — 도메인 개념이다"

도메인을 코드의 시민으로 끌어올리면, 코드가 비즈니스 언어로 말하기 시작한다.

### 관련 노트

- [[../../language/java/record]] — Java에서 VO를 만드는 사실상의 표준 도구
