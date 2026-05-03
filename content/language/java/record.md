---
title: Java Record — 불변 값 객체를 한 줄로
type: concept
tags: [java, record, immutability, value-object, ddd]
related:
  - "[[sealed-interface]]"
  - "[[../../architecture/ddd/value-object]]"
last_reviewed: 2026-05-03
publish: true
---

# Java Record — 불변 값 객체를 한 줄로

## 목차

- [한 줄 정의](#한-줄-정의)
- [왜 record가 필요한가](#왜-record가-필요한가)
- [record의 정체](#record의-정체)
- [헤더 괄호의 의미](#헤더-괄호의-의미)
- [컴파일러가 자동 생성하는 것](#컴파일러가-자동-생성하는-것)
- [Canonical Constructor](#canonical-constructor)
- [Compact Canonical Constructor](#compact-canonical-constructor)
- [record의 제약](#record의-제약)
- [메서드 추가하기](#메서드-추가하기)
- [accessor 오버라이드의 함정](#accessor-오버라이드의-함정)
- [가변 컴포넌트의 함정](#가변-컴포넌트의-함정)
- [Lombok과의 비교](#lombok과의-비교)
- [핵심 체크리스트](#핵심-체크리스트)

## 한 줄 정의

**"불변 값 객체(Value Object)를 만들기 위한 특수한 클래스 선언"**. Java 16+ 정식 도입. 컴파일러가 필드, 생성자, accessor, equals/hashCode/toString을 한 줄 선언으로부터 자동 생성한다.

```java
public record EventId(UUID value) { }
```

이 한 줄이 30줄 분량의 보일러플레이트와 동치.

## 왜 record가 필요한가

### Primitive obsession 방지

> **Primitive obsession** — "원시 타입 집착". 도메인의 의미 있는 개념을 String/UUID/Long 같은 범용 타입으로만 표현하는 안티패턴. 타입이 너무 넓어서 컴파일러가 의미적 실수를 잡지 못한다.

같은 `UUID` 타입이면 그게 사용자 ID인지 이벤트 ID인지 컴파일러는 구분할 수 없다. 그래서 인자 순서 실수 같은 버그는 **운영에 나가서야 발견**된다. 도메인 개념마다 별도 타입으로 좁히면 같은 실수가 **컴파일 시점에** 잡힌다.

**❌ UUID를 그대로 쓰면**

```java
void cancel(UUID eventId, UUID userId) { ... }

UUID userId  = UUID.fromString("aaaa-...");
UUID eventId = UUID.fromString("bbbb-...");

cancel(userId, eventId);
//     ↑ UUID    ↑ UUID
// 인자 순서가 바뀌었지만 타입이 둘 다 UUID라서 컴파일러는 구분 못 함
// → 컴파일 통과, 운영에서 엉뚱한 이벤트 취소
```

컴파일러 입장에선 `cancel(UUID, UUID)`에 `(UUID, UUID)`를 넘긴 거라 완벽히 합법. 의미적 실수를 알 길이 없음.

**✅ record VO로 감싸면**

```java
public record EventId(UUID value) { }
public record UserId(UUID value) { }

void cancel(EventId eventId, UserId userId) { ... }

UserId  userId  = new UserId(UUID.fromString("aaaa-..."));
EventId eventId = new EventId(UUID.fromString("bbbb-..."));

cancel(userId, eventId);
//     ↑ UserId  ↑ EventId
// ❌ error: incompatible types: UserId cannot be converted to EventId
//   (첫 자리는 EventId를 요구하는데 UserId가 들어옴)
```

`EventId`와 `UserId`는 둘 다 UUID 하나만 들고 있지만 **서로 상속 관계도 없는 무관한 두 타입**. 값이 같아도 타입이 다르면 호환 불가 → 컴파일러가 인자 순서 실수를 잡아줌.

```java
EventId e = new EventId(uuid);
UserId  u = e;                       // ❌ 컴파일 에러
UserId  u = new UserId(e.value());   // ✅ 명시적 변환만 허용
```

### 보일러플레이트 지옥 해소

전통 Java로 VO를 만들면 필드 + 생성자 + getter + equals + hashCode + toString을 일일이 손으로 써야 함. record는 한 줄로 끝.

## record의 정체

`record`는 메서드가 아니라 **클래스 선언의 일종**. `class`, `interface`, `enum`과 같은 레벨의 키워드.

```
class      ────►  일반 클래스 (가변, 상속 가능)
record     ────►  불변 데이터 클래스 (final, 값 동등성)
enum       ────►  열거형
interface  ────►  인터페이스
```

내부적으로는 `java.lang.Record`를 자동 상속하는 **final class**로 컴파일됨.

### 선언 가능한 위치

`class`나 `interface`를 선언할 수 있는 모든 위치에 가능.

- **Top-level**: 파일 단위
- **Nested**: 다른 클래스 안 (자동 static)
- **Local**: 메서드 안 (Java 16+, 스트림 파이프라인 임시 튜플로 유용)
- **Interface 내부**: 자동 static

## 헤더 괄호의 의미

일반 class와 시각적으로 다른 점 — record는 **클래스 헤더에 괄호**가 등장한다.

```java
public record EventId(UUID value) { }
//                   ↑ 클래스 헤더의 괄호
```

이 괄호는 메서드 매개변수가 아니라 **컴포넌트(component) 목록**. 한 번에 세 가지 역할을 한다:

| 역할 | 변환되는 코드 |
|---|---|
| 필드 선언 | `private final UUID value;` |
| 생성자 파라미터 | `public EventId(UUID value)` |
| accessor 이름 힌트 | `public UUID value()` |

## 컴파일러가 자동 생성하는 것

```java
public record EventId(UUID value) { }
```

위 한 줄은 다음과 거의 동치인 코드로 컴파일됨:

```java
public final class EventId extends java.lang.Record {
    private final UUID value;

    public EventId(UUID value) {
        this.value = value;
    }

    public UUID value() { return this.value; }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof EventId other)) return false;
        return Objects.equals(this.value, other.value);
    }

    @Override
    public int hashCode() {
        return Objects.hash(value);
    }

    @Override
    public String toString() {
        return "EventId[value=" + value + "]";
    }
}
```

자동 생성되는 것 정리:

1. `private final` 필드 (컴포넌트마다 하나씩)
2. **canonical constructor** (모든 컴포넌트를 받는 생성자)
3. **accessor 메서드** (컴포넌트 이름과 동일 — `value()` 형태, `getValue()` 아님)
4. `equals` (모든 필드 비교)
5. `hashCode` (모든 필드 기반)
6. `toString` (`EventId[value=...]` 형태)
7. JVM 차원의 record 메타데이터 (Record attribute, RecordComponents)

## Canonical Constructor

**헤더에 선언한 모든 컴포넌트를 같은 순서, 같은 타입으로 받는 정식 생성자.** record엔 항상 존재한다 (안 적어도 자동).

```java
public record User(Long id, String name, Email email) { }
// canonical constructor: public User(Long id, String name, Email email)
```

### 직접 작성도 가능 — 검증/정규화 끼워넣기

```java
public record EventId(UUID value) {
    public EventId(UUID value) {
        Objects.requireNonNull(value, "value");
        this.value = value;
    }
}
```

### 추가 생성자는 canonical을 호출해야 함

```java
public record EventId(UUID value) {
    public EventId() {
        this(UUID.randomUUID());          // canonical 호출 필수
    }

    public EventId(String s) {
        this(UUID.fromString(s));         // canonical 호출 필수
    }
}
```

이유: 모든 필드 초기화는 canonical을 통해서만 일어나야 한다는 규칙. 불변성 보장 장치.

## Compact Canonical Constructor

**record 전용의 축약 생성자 문법**. 파라미터 리스트와 필드 할당을 컴파일러가 자동으로 채워준다.

```java
public record Email(String address) {
    public Email {                                  // ← 괄호 없음
        Objects.requireNonNull(address);
        if (!address.contains("@"))
            throw new IllegalArgumentException();
        address = address.trim().toLowerCase();     // ← 정규화
        // this.address = address 는 자동
    }
}
```

컴파일러가 풀어주는 모습:

```java
public Email(String address) {                     // 파라미터 자동
    Objects.requireNonNull(address);
    if (!address.contains("@"))
        throw new IllegalArgumentException();
    address = address.trim().toLowerCase();
    this.address = address;                        // 자동 할당
}
```

### Compact form의 핵심 규칙

- 본문에서 만지는 `value`는 **파라미터(로컬 변수)**, 필드가 아님
- `this.value = ...`를 직접 쓰면 컴파일 에러 (final 필드 이중 할당)
- **검증 + 정규화에만 집중**, 할당은 컴파일러에 맡김

## record의 제약

### 1. 자동 final → 다른 클래스가 상속 불가

record는 선언만 해도 자동으로 `final`. 다른 클래스가 `extends`로 상속 시도하면 컴파일 에러.

```java
public record EventId(UUID value) { }

public class SpecialId extends EventId { }   // ❌ cannot inherit from final
```

**진짜 이유**: 자식이 필드를 추가하면 equals/hashCode 대칭성이 깨짐.

```java
// 만약 상속이 가능했다면 — 가정
record Point(int x, int y) { }
class ColorPoint extends Point { String color; }

Point p = new Point(1, 2);
ColorPoint c = new ColorPoint(1, 2, "red");
p.equals(c);   // (x,y) 같으니까 true?
c.equals(p);   // color 다르니까 false?
               // → 대칭성 깨짐 = equals 계약 위반
```

이걸 원천 차단하려고 record는 언어 차원에서 상속을 금지.

### 2. record가 다른 class를 상속 불가

이미 `java.lang.Record`를 상속 중이라 부모 자리가 점유됨 (Java는 단일 상속).

```java
public record EventId(UUID value) extends BaseId { }   // ❌ 문법 에러
```

공통 동작 공유는 **인터페이스 + default method**로:

```java
public interface Identifiable {
    UUID value();
    default String asString() { return value().toString(); }
}

public record EventId(UUID value) implements Identifiable { }
public record UserId(UUID value)  implements Identifiable { }
```

### 3. 인스턴스 필드 추가 불가

헤더에 선언한 컴포넌트만 인스턴스 필드. 본문에 새 필드 추가 못 함 (static 상수는 OK).

```java
public record EventId(UUID value) {
    private String cache;                                // ❌ 컴파일 에러
    private static final EventId NIL = new EventId(...); // ✅ static OK
}
```

이유: "값(value)"이라는 정체성을 강제. 상태가 늘어나면 더 이상 단순 값이 아님.

### 4. 인터페이스 구현은 가능

```java
public record User(Long id, String name)
    implements Identifiable, Comparable<User>, Serializable {

    @Override
    public int compareTo(User other) {
        return this.id.compareTo(other.id);
    }
}
```

자동 생성되는 accessor가 인터페이스 메서드 시그니처와 일치하면 추가 코드 0줄로 구현됨. **sealed interface + record 조합으로 sum type**을 만들 수 있는 토대가 이 규칙.

### 제약 요약표

| 제약 | 가능? | 이유 |
|---|---|---|
| 다른 클래스가 record 상속 | ❌ | 자동 final (equals 계약 보호) |
| record가 다른 class 상속 | ❌ | Record가 이미 부모 (단일 상속) |
| record가 인터페이스 구현 | ✅ | 다중 구현 허용 |
| 인스턴스 필드 추가 | ❌ | 값 객체 정체성 강제 |
| static 필드/메서드 추가 | ✅ | 인스턴스 상태와 무관 |
| 인스턴스 메서드 추가 | ✅ | 도메인 행위 표현 |

## 메서드 추가하기

record body에 자유롭게 메서드 정의 가능. "데이터 그릇"에 그치지 않고 **DDD의 Value Object**로 격상하는 핵심.

### 파생 값 계산

```java
public record DateRange(LocalDate start, LocalDate end) {
    public long days() {
        return ChronoUnit.DAYS.between(start, end);
    }

    public boolean contains(LocalDate date) {
        return !date.isBefore(start) && !date.isAfter(end);
    }
}
```

### Wither 패턴 (불변 변경)

setter 대신 "변경된 새 인스턴스 반환".

```java
public record User(Long id, String name, String email) {
    public User withEmail(String newEmail) {
        return new User(id, name, newEmail);
    }
}
```

### 도메인 행위

```java
public record Money(BigDecimal amount, Currency currency) {
    public Money add(Money other) {
        if (!currency.equals(other.currency))
            throw new IllegalArgumentException("currency mismatch");
        return new Money(amount.add(other.amount), currency);   // 새 객체
    }

    public boolean isZero() {
        return amount.signum() == 0;
    }
}
```

핵심: **값을 바꾸는 게 아니라, 바뀐 값을 가진 새 객체를 만든다.** 모든 불변 객체의 공통 사고방식 (`String.toUpperCase()`, `BigDecimal.add()`, `LocalDate.plusDays()` 모두 같은 패턴).

### 정적 팩토리

```java
public record EventId(UUID value) {
    public static EventId generate() {
        return new EventId(UUID.randomUUID());
    }

    public static EventId from(String s) {
        return new EventId(UUID.fromString(s));
    }
}
```

### 좋은 메서드 vs 나쁜 메서드

| 좋은 메서드 | 나쁜 메서드 |
|---|---|
| 컴포넌트만으로 계산 | 외부 상태(DB, 시간 등) 의존 |
| 불변 (새 인스턴스 반환 / 순수 계산) | 부수효과 있음 |
| 도메인 의미를 표현 | 영속성/CRUD 로직 |
| `days()`, `withEmail()` | `save()`, `delete()`, `sendEmail()` |

## accessor 오버라이드의 함정

자동 생성된 `equals`, `hashCode`, `toString`은 **accessor 메서드를 거치지 않고 필드를 직접 읽는다**. 그래서 accessor를 변형하는 식으로 오버라이드하면 "보이는 값"과 "비교되는 값"이 어긋난다.

```java
public record Name(String value) {
    @Override
    public String value() {
        return value.trim();   // accessor만 trim
    }
}

Name a = new Name("alice");        // 필드: "alice"
Name b = new Name("alice  ");      // 필드: "alice  "

a.value();         // "alice"
b.value();         // "alice"      ← 같아 보이는데
a.equals(b);       // false        ← 어긋남
```

자동 생성된 `equals`는 (개념적으로):

```java
public boolean equals(Object o) {
    if (!(o instanceof Name other)) return false;
    return Objects.equals(this.value, other.value);
    //                    ↑ 필드 직접 접근 (메서드 호출 X)
}
```

**`this.value` ≠ `this.value()`** 가 될 수 있고, equals는 전자(필드)를 본다.

### 왜 컴파일러는 필드를 직접 읽도록 설계했나

1. **성능**: equals/hashCode가 컬렉션 키로 빈번히 호출됨 → 메서드 호출 오버헤드 회피
2. **신뢰성**: 사용자의 accessor 오버라이드가 equals 계약을 깨지 못하도록 분리
3. **사양**: JLS가 record equals를 컴포넌트 필드 직접 비교로 명시

### 해결: compact constructor에서 정규화

```java
public record Name(String value) {
    public Name {
        value = value.trim();   // 필드 자체를 정규화
    }
}

Name a = new Name("alice");
Name b = new Name("alice  ");
a.equals(b);   // ✅ true (둘 다 필드값이 "alice")
```

**Single Source of Truth**: 값의 진실은 한 곳(필드)에만 두고, 모든 표현이 거기서 파생되도록.

### accessor 오버라이드가 안전한 케이스

값을 변형하지 않고 그대로 반환하는 한 OK.

```java
public record Tags(List<String> values) {
    @Override
    public List<String> values() {
        return List.copyOf(values);   // 같은 내용, 외부 수정만 차단
    }
}
```

## 가변 컴포넌트의 함정

record 컴포넌트가 가변 객체(List, Date, 배열)면 외부에서 수정해 record의 "불변성"이 깨질 수 있음.

```java
public record Tags(List<String> values) { }

List<String> list = new ArrayList<>(List.of("a", "b"));
Tags t1 = new Tags(list);

list.add("c");          // 외부에서 list 변경
t1.values();            // [a, b, c]   ← record가 바뀐 것처럼 보임
```

### 해결: compact constructor에서 방어적 복사

```java
public record Tags(List<String> values) {
    public Tags {
        values = List.copyOf(values);   // 입력 시점에 불변 복사
    }
}
```

`List.copyOf`는 입력 내용을 복사한 **새 불변 리스트**를 반환. 원본을 수정해도 record는 영향 없음.

### `==` 와 `.equals()` 구분

방어적 복사를 해도 `equals`는 정상 동작한다는 점이 중요.

```java
Tags t1 = new Tags(list);
Tags t2 = new Tags(list);

t1.values() == t2.values();         // false (다른 인스턴스)
t1.values().equals(t2.values());    // true  (내용 같음)
t1.equals(t2);                      // true  (record equals = 내용 비교)
```

| 비교 | 의미 |
|---|---|
| `==` | 같은 객체 인스턴스인가? (참조 동등성) |
| `.equals()` | 내용이 같은가? (값 동등성) |

`List.equals`, `String.equals`, record의 자동 `equals` 모두 **값 동등성**. 다른 인스턴스여도 내용 같으면 true.

### 배열 컴포넌트 주의

배열의 `.equals()`는 참조 비교(`==`와 동치)라 record equals가 의도대로 작동하지 않는다.

```java
public record BadTags(String[] values) { }

BadTags a = new BadTags(new String[]{"x", "y"});
BadTags b = new BadTags(new String[]{"x", "y"});
a.equals(b);   // ❌ false — 배열 equals는 참조 비교
```

→ record 컴포넌트로는 **배열보다 List** 권장.

## Lombok과의 비교

record를 Lombok 어노테이션으로 매핑하면:

```
record  ≈  @Value
        =  @AllArgsConstructor
         + @Getter
         + @EqualsAndHashCode
         + @ToString
         + 필드 final
         + 클래스 final
```

`@Value`가 가장 가까운 짝. 단, 차이점:

| | Lombok `@Value` | record |
|---|---|---|
| 동작 시점 | 어노테이션 처리기 (빌드 시 코드 생성) | 언어/컴파일러 자체 |
| 메타데이터 | 일반 class로 보임 | JVM 레벨에서 record로 식별됨 |
| 인스턴스 필드 추가 | 가능 | 불가 (헤더가 전부) |
| accessor 이름 | `getValue()` | `value()` |
| 라이브러리 의존 | Lombok 필요 | Java 표준 |

`@RequiredArgsConstructor`는 "final + @NonNull 필드만"이라서 record canonical constructor와 의미가 다름. record의 canonical constructor는 **모든 컴포넌트** 대상이므로 `@AllArgsConstructor`에 가깝다.

## 핵심 체크리스트

> **Record란?**
>
> 1. **불변 값 객체**를 만드는 클래스 선언 (class의 특수 형태)
> 2. **헤더의 컴포넌트** = final 필드 + 생성자 파라미터 + accessor를 한 번에 선언
> 3. 컴파일러가 자동 생성: **canonical constructor / accessor(`컴포넌트명()`) / equals / hashCode / toString**
> 4. **인스턴스 필드 추가 불가** (헤더가 전부) — 진짜 불변 보장
> 5. 자동 **final** → 상속 불가 / `extends` 불가, 단 `implements`는 가능
> 6. **compact canonical constructor**로 검증·정규화 끼워넣기 가능

### 함정 회피 룰

- 값 변형(trim, lowercase, scale 등)은 **compact constructor에서 필드 자체에** 적용. accessor 오버라이드로 우회하지 말 것.
- 가변 컴포넌트(List, Date 등)는 **compact constructor에서 방어적 복사** (`List.copyOf` 등).
- 배열보다 List 사용 (배열 equals는 참조 비교).
- 부수효과/외부 의존 메서드는 record가 아니라 별도 서비스/리포지토리에.

### 사고 전환

> **가변 객체 사고**: "이 객체의 상태를 바꾼다"
> **불변 객체 사고**: "이 객체를 입력으로 받아, 새 객체를 만들어 반환한다"

`String.toUpperCase()`, `BigDecimal.add()`, `LocalDate.plusDays()`가 모두 같은 패턴. record로 도메인 모델을 만들면 자연스럽게 함수형 스타일이 몸에 밴다.
