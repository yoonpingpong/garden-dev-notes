---
title: VO에 도메인 행위를 응집하기
type: example
tags: [ddd, value-object, domain-modeling, java]
related:
  - "[[value-object]]"
  - "[[../../language/java/record]]"
last_reviewed: 2026-05-25
publish: true
---

# VO에 도메인 행위를 응집하기

## 목차

- [개요](#개요)
- [시나리오 — 주문 합계 계산](#시나리오--주문-합계-계산)
  - [요구사항](#요구사항)
  - [❌ 절차적 접근](#-절차적-접근)
  - [✅ VO 기반 접근](#-vo-기반-접근)
  - [무엇이 좋아졌나](#무엇이-좋아졌나)
  - [도메인 언어로 읽힌다는 것](#도메인-언어로-읽힌다는-것)
- [관련 노트](#관련-노트)

## 개요

값과 관련된 로직을 **그 값의 타입에 두면**, 코드가 도메인 언어로 읽힌다. 같은 비즈니스 규칙을 절차적으로 풀어쓴 코드와 VO에 응집한 코드를 비교해 그 차이를 본다.

전체 맥락은 [[value-object]] 참조. 이 노트는 그중 **"도메인 행위의 자연스러운 위치"** 항목의 풀 시나리오.

## 시나리오 — 주문 합계 계산

### 요구사항

- 주문(Order)에는 여러 상품(LineItem)이 들어있고, 각 상품마다 가격(price)이 있다.
- 모든 상품 가격을 합한다.
- 합계가 임계값(`THRESHOLD`)을 넘으면 할인율(`DISCOUNT_RATE`)을 곱해 할인된 금액을 돌려준다.

### ❌ 절차적 접근

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

### ✅ VO 기반 접근

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

### 무엇이 좋아졌나

| | 절차적 (`OrderService`) | VO 기반 (`Order` + `Money`) |
|---|---|---|
| 합산 로직 위치 | OrderService 안의 BigDecimal for-loop | `Money.add` (값의 타입에) |
| 할인 로직 위치 | OrderService 안의 if + multiply | `Order.total()` (도메인 객체에) |
| 통화 안전성 | 없음 (KRW/USD 섞여도 통과) | `Money.add`가 통화 검증 |
| 재사용성 | OrderService 메서드 호출 필요 | `order.total()` 한 줄 |
| 가독성 | 산술 연산 나열 | `sum.isGreaterThan(threshold)` ← 도메인 언어 |

### 도메인 언어로 읽힌다는 것

같은 비즈니스 로직을 두 버전의 코드로 작성하고, 각각을 한국어로 소리내어 읽어본다.

#### VO 버전 — `Order.total()`

```java
Money sum = items.stream()
    .map(LineItem::price)
    .reduce(Money.ZERO_KRW, Money::add);
return sum.isGreaterThan(threshold)
    ? sum.multiply(discountRate)
    : sum;
```

> "items의 가격들을 0원에서부터 더해서 합계를 만든다.
> 합계가 임계값보다 크면 할인율을 곱한 값을, 아니면 합계 그대로 돌려준다."

→ 비즈니스 담당자가 요구사항을 말할 때 쓰는 **그대로의 문장**.

#### 절차적 버전 — `OrderService.calculateTotal()`

```java
BigDecimal sum = BigDecimal.ZERO;
for (BigDecimal p : prices) sum = sum.add(p);
if (sum.compareTo(THRESHOLD) > 0) sum = sum.multiply(DISCOUNT_RATE);
return sum;
```

> "BigDecimal을 ZERO에서 시작해서, for로 add하고,
> compareTo가 0보다 크면 multiply한 결과를 돌려준다."

→ 개발자가 머릿속에서 **산술을 따라가야** 이해되는 문장.

#### 핵심 차이

값의 타입(`Money`)에 행위(`add`, `multiply`, `isGreaterThan`)를 응집하면 세 가지가 바뀐다:

- 호출부의 메서드 이름이 **비즈니스 동사**가 됨 — `add`, `isGreaterThan`
- 산술 연산자(`compareTo > 0`)가 사라지고 **의도가 메서드명에 박힘**
- 코드 한 줄 ≒ 비즈니스 문장 한 줄

이게 VO가 *"도메인 행위의 자연스러운 위치"* 인 이유.

## 관련 노트

- [[value-object]] — VO 개념과 5가지 핵심 용도 (이 시나리오의 상위 문서)
- [[../../language/java/record]] — Java에서 VO를 만드는 도구 (`Money` 같은 불변 값 객체 구현)
- [[immutability-benefits]] — VO의 또 다른 효과 (불변성으로 안전한 공유)
