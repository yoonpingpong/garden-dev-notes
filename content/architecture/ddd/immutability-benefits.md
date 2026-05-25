---
title: VO 불변성이 가져오는 안전성
type: example
tags: [ddd, value-object, immutability, concurrency, java]
related:
  - "[[value-object]]"
  - "[[../../language/java/record]]"
  - "[[../../language/java/object-comparison]]"
last_reviewed: 2026-05-25
publish: false
---

# VO 불변성이 가져오는 안전성

## 목차

- [개요](#개요)
- [시나리오 — 같은 Address를 두 서비스에 넘김](#시나리오--같은-address를-두-서비스에-넘김)
  - [❌ 가변 클래스](#-가변-클래스)
  - [✅ VO](#-vo)
  - [불변성에서 파생되는 3가지 보너스](#불변성에서-파생되는-3가지-보너스)
- [핵심 원리](#핵심-원리)
- [관련 노트](#관련-노트)

## 개요

객체를 여러 곳에서 참조할 때, **그 객체가 바뀔 수 있느냐 없느냐**가 코드의 복잡도를 좌우한다. 가변 객체는 "누가 언제 바꿨지?"라는 추적 지옥을 만들고, 불변 객체는 그 질문 자체를 없앤다.

전체 맥락은 [[value-object]] 참조. 이 노트는 그중 **"불변성으로 안전한 공유"** 항목의 풀 시나리오.

## 시나리오 — 같은 Address를 두 서비스에 넘김

배송 라벨용으로 받은 주소 하나를, 청구서 발행과 배송 라우팅에 동시에 사용한다고 해보자.

### ❌ 가변 클래스

```java
class Address {
    String street;
    String city;
    String zip;

    public void setCity(String c) { this.city = c; }
    public String getCity() { return city; }
    // 나머지 getter/setter 생략
}

class BillingService {
    public void issue(Address address) {
        // 청구서 형식에 맞추려고 도시명 대문자로 정규화
        address.setCity(address.getCity().toUpperCase());   // ⚠️ 원본을 바꿔버림
        // 정규화된 address로 청구서를 인쇄한다 (이후 코드 생략)
    }
}

class ShippingService {
    // 도시명 → 배송 권역 매핑 (원본 도시명을 키로 사용)
    private static final Map<String, String> CITY_ZONES = Map.of(
        "Seoul", "ZONE-A",
        "Busan", "ZONE-B"
    );

    public void route(Address address) {
        // 도시명을 키로 배송 권역을 조회
        String zone = CITY_ZONES.get(address.getCity());
        // address.getCity() == "SEOUL" → CITY_ZONES.get("SEOUL") → null
        // → 매핑 실패, 라우팅 불가
    }
}

// 사용처
Address shared = new Address("Teheran-ro", "Seoul", "06234");

billingService.issue(shared);     // 안에서 city를 "SEOUL"로 바꿈
shippingService.route(shared);    // shared.city == "SEOUL" → 라우팅 깨짐
```

문제점:
- `BillingService`가 인자로 받은 `Address`를 **수정**해버려서 `ShippingService`까지 오염
- 호출자 입장에선 `shared`를 그냥 넘겼을 뿐인데 값이 바뀌어있음 → "어느 서비스에서 바꿨지?" 디버깅 시작
- 멀티스레드면 상황은 훨씬 더 나쁨 — 한 스레드가 읽는 도중 다른 스레드가 쓰면 race condition

### ✅ VO

```java
public record Address(String street, String city, String zip) { }
//                    ↑ 모든 컴포넌트 자동 final, setter 자체가 없음

class BillingService {
    public void issue(Address address) {
        // 정규화가 필요하면 새 인스턴스를 만든다 (원본 보존)
        Address normalized = new Address(
            address.street(),
            address.city().toUpperCase(),
            address.zip()
        );
        // normalized로 청구서를 인쇄한다 (이후 코드 생략)
        // address(원본)는 건드리지 않음
    }
}

class ShippingService {
    // 도시명 → 배송 권역 매핑 (원본 도시명을 키로 사용)
    private static final Map<String, String> CITY_ZONES = Map.of(
        "Seoul", "ZONE-A",
        "Busan", "ZONE-B"
    );

    public void route(Address address) {
        // 도시명을 키로 배송 권역을 조회
        String zone = CITY_ZONES.get(address.city());
        // address.city() == "Seoul" → CITY_ZONES.get("Seoul") → "ZONE-A"
        // → 매핑 성공, 라우팅 정상 ✅
    }
}

// 사용처
Address shared = new Address("Teheran-ro", "Seoul", "06234");

billingService.issue(shared);     // shared는 절대 안 바뀜
shippingService.route(shared);    // shared.city() == "Seoul" → ZONE-A ✅
```

`Address`는 record라 setter가 아예 없음 → `BillingService`가 원본을 수정하려 해도 **컴파일 자체가 불가능**. 변경이 필요하면 `new Address(...)`로 새 인스턴스를 만들 수밖에 없고, 그 새 인스턴스는 호출자의 `shared`와 무관함. → `ShippingService`가 받는 `address.city()`는 항상 원본 `"Seoul"` → 매핑 성공.

### 불변성에서 파생되는 3가지 보너스

> 아래 세 예제는 모두 `Address`가 위에서 정의한 **record 버전(VO)** 이라는 전제 하에 성립한다.
> 만약 가변 `class Address`였다면 셋 다 깨진다.

#### (1) 스레드 안전 — 락 불필요

```java
// Address는 record (불변) → 한 번 만들어진 뒤로는 절대 안 바뀜
Address shared = new Address("Teheran-ro", "Seoul", "06234");

// 100개 스레드가 동시에 shared를 읽어도 안전
ExecutorService pool = Executors.newFixedThreadPool(100);
for (int i = 0; i < 100; i++) {
    pool.submit(() -> processInThread(shared));   // 락/synchronized 불필요
}
```

**전제**: `Address`가 불변이라 어떤 스레드도 `shared`를 *수정할 방법이 없다*. 100개 스레드가 동시에 `shared.city()`를 읽어도 모두 같은 `"Seoul"`을 받는다 — 경쟁 자체가 성립하지 않음.

**가변이면 어떻게 되나** (비교):

```java
// 만약 Address가 가변 class였다면
class Address { String city; public void setCity(String c) { ... } }

Address shared = new Address(...);
pool.submit(() -> shared.setCity("X"));   // 스레드 A: 쓰기
pool.submit(() -> read(shared.getCity())); // 스레드 B: 읽기
// → race condition. 락(synchronized/ReadWriteLock) 없이는 안전 보장 불가
```

값이 바뀌지 않으니 동시 읽기에서 불일치가 발생할 여지 자체가 없다. Java 동시성에서 가장 안전한 객체는 **항상 불변 객체**.

#### (2) 캐시 키로 안전

```java
Map<Address, DeliveryFee> feeCache = new HashMap<>();

Address addr = new Address("Teheran-ro", "Seoul", "06234");
feeCache.put(addr, new DeliveryFee(3000));

// 만약 가변이고 누군가 addr.setCity("Busan")으로 바꾸면?
// → addr의 hashCode가 변경됨
// → feeCache에서 더 이상 찾을 수 없음 (메모리 누수 + 잃어버린 데이터)

// VO(record)면 city() 자체를 못 바꿈 → 캐시 영원히 유효
feeCache.get(new Address("Teheran-ro", "Seoul", "06234"));   // 정상 조회
```

`HashMap`은 키를 넣을 때의 hashCode를 기억하고 있다. 키가 가변이고 외부에서 변형되면 **맵 안에서 영영 못 찾는 유령 객체**가 됨. 불변이면 이 위험 자체가 없다. (관련: [[../../language/java/object-comparison#함정-5--가변-객체를-hashmap-키로|object-comparison: 가변 객체를 HashMap 키로]])

#### (3) equals/hashCode가 일생 동안 일관됨

```java
// Address는 record (불변) → 필드를 바꿀 방법이 없음
Address a = new Address("Teheran-ro", "Seoul", "06234");
int hashAtStart = a.hashCode();

// ... 천 번의 함수 호출, 다른 서비스 통과, 다른 스레드 노출 ...

a.hashCode() == hashAtStart;   // 항상 true (필드가 안 바뀌니 hashCode도 안 바뀜)
```

**전제**: 불변이라 hashCode 계산의 입력(필드)이 절대 변하지 않음 → 출력(hashCode)도 평생 같음. 가변이었다면 누군가 `setCity(...)`를 부르는 순간 hashCode가 바뀌어 (2)의 캐시 키 사고로 직결.

## 핵심 원리

> **가변이면**: "이 객체가 지금 어떤 상태인지" 매 시점 확인해야 함. 코드 추론이 시간축에 의존.
> **불변이면**: 한 번 만들어진 값은 끝까지 그 값. 코드 추론이 **국소적** — 이 줄만 보면 됨.

함수형 언어가 일찍이 발견한 진실 — *"상태를 바꾸지 않으면 버그의 절반이 사라진다"*. VO는 객체지향 안에서 이 원칙을 실천하는 도구.

## 관련 노트

- [[value-object]] — VO 개념과 5가지 핵심 용도 (이 시나리오의 상위 문서)
- [[../../language/java/record]] — Java에서 불변 VO를 만드는 도구
- [[../../language/java/object-comparison]] — equals/hashCode와 가변 객체의 함정
- [[domain-behavior-in-vo]] — VO의 또 다른 효과 (값에 도메인 행위 응집)
