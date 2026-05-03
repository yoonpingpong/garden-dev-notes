---
title: Clean Architecture Overview
type: concept
tags: [architecture, clean-architecture, dependency-rule, ddd, hexagonal]
related: []
last_reviewed: 2026-05-03
publish: false
---

# Clean Architecture Overview

## 목차

- [한 줄 정의](#한-줄-정의)
- [등장 배경](#등장-배경)
- [핵심 원칙 — The Dependency Rule](#핵심-원칙--the-dependency-rule)
- [4개 동심원](#4개-동심원)
- [Spring Boot 패키지 매핑](#spring-boot-패키지-매핑)
- [의존성 위반의 신호](#의존성-위반의-신호)
- [명시적 의존 vs 암묵적 결합](#명시적-의존-vs-암묵적-결합)
- [의존성 분리가 가져오는 실질 이득](#의존성-분리가-가져오는-실질-이득)
- [함정 / 자주 하는 오해](#함정--자주-하는-오해)
- [사고 모델](#사고-모델)
- [참고](#참고)

## 한 줄 정의

비즈니스 규칙을 외부 기술(DB, 웹 프레임워크, 외부 API)로부터 격리시키기 위한 **계층형 아키텍처 패턴**. Robert C. Martin(Uncle Bob)이 2012년 제안.

> ⚠️ **자주 하는 오해**: "Clean Architecture는 폴더 구조다" — **아니다.**
> Clean Architecture의 본질은 **의존성 방향(Dependency Rule)**. 폴더 구조는 그 결과물일 뿐이다.

## 등장 배경

전통적 계층 아키텍처(Web → Service → Repository)의 문제:

- **DB 변경이 비즈니스 로직에 전파됨** — JPA 어노테이션이 도메인 객체에 박혀 있음
- **테스트가 무거움** — 단위 테스트마다 Spring Context, DB 필요
- **프레임워크에 종속됨** — Spring을 빼면 도메인이 동작 안 함
- **비즈니스 가독성 낮음** — 핵심 규칙이 인프라 코드에 가려짐

Uncle Bob은 이 문제들의 근원을 **"의존성 방향이 잘못되어 있다"** 로 진단했다. 비즈니스(안쪽)가 인프라(바깥쪽)를 의존하면, 인프라 변경이 비즈니스를 흔든다. 의존성을 **반대로 뒤집으면** 해결된다.

## 핵심 원칙 — The Dependency Rule

> **의존성은 항상 안쪽으로만 향해야 한다. 안쪽 레이어는 바깥쪽 레이어를 알아서는 안 된다.**

이 한 문장이 Clean Architecture의 전부다. 나머지는 모두 이 규칙의 적용일 뿐.

```
바깥                                                                안쪽
 ──────────────────────────────────────────────────────────────────→
 Frameworks  →  Adapters  →  Application  →  Domain
 (Spring,       (Controller,   (UseCase,        (Entity,
  JPA,           JPA Adapter,   Port,            Value Object,
  Web Server)    Meta Adapter)  Service)         Domain Service)

 의존성 화살표는 모두 안쪽으로만 향함 (← ← ← ←)

 ✅ Adapter가 Application을 의존
 ✅ Application이 Domain을 의존
 ❌ Domain이 Application을 의존 (위반)
 ❌ Application이 Adapter를 의존 (위반)
```

### 의존성 역전 원리(DIP)와의 관계

Application이 외부 시스템(DB, 외부 API)을 호출해야 하는데도 Adapter를 의존하지 않는 이유는 **Application이 Port(인터페이스)를 정의하고, Adapter가 그 Port를 구현**하기 때문이다.

```
[Application]
  └─ EventStorePort (interface)   ← Application이 정의
                ↑
                │ 구현
                │
[Adapter]
  └─ EventStorePersistenceAdapter ← Adapter가 구현
```

→ **소유권은 안쪽에, 구현은 바깥쪽에**. 이게 의존성 역전(Dependency Inversion Principle).

## 4개 동심원

Uncle Bob의 원본 다이어그램:

```
        ┌──────────────────────────────────────────┐
        │  ⓪ Frameworks & Drivers                  │
        │   (Spring, JPA, Web Server, DB)           │
        │  ┌────────────────────────────────────┐  │
        │  │  ① Interface Adapters               │  │
        │  │   (Controllers, Presenters,         │  │
        │  │    Gateways, Mappers)               │  │
        │  │  ┌──────────────────────────────┐  │  │
        │  │  │  ② Application Business Rules │  │  │
        │  │  │   (Use Cases)                  │  │  │
        │  │  │  ┌────────────────────────┐   │  │  │
        │  │  │  │  ③ Enterprise          │   │  │  │
        │  │  │  │     Business Rules     │   │  │  │
        │  │  │  │     (Entities)          │   │  │  │
        │  │  │  └────────────────────────┘   │  │  │
        │  │  └──────────────────────────────┘  │  │
        │  └────────────────────────────────────┘  │
        └──────────────────────────────────────────┘
```

### 각 원의 책임

| 원 | 이름 | 책임 | 의존하는 것 |
| --- | --- | --- | --- |
| ③ | **Enterprise Business Rules** (Entities) | 가장 핵심적인 비즈니스 규칙. 회사가 망해도 살아남을 규칙 | 없음 (가장 안쪽) |
| ② | **Application Business Rules** (Use Cases) | 이 애플리케이션 특유의 비즈니스 흐름 | ③만 |
| ① | **Interface Adapters** | 외부 ↔ 내부 데이터 변환 (Controller, Mapper, Adapter) | ②, ③ |
| ⓪ | **Frameworks & Drivers** | 프레임워크 자체, DB 자체, 외부 시스템 | ①, ②, ③ (간접) |

### 핵심 직관

- **③ Domain**: *"주문은 0원 미만일 수 없다"* 같은 회사의 본질적 규칙
- **② Application**: *"주문이 들어오면 재고를 차감하고 결제 처리 후 알림을 보낸다"* 같은 유스케이스
- **① Adapter**: *"HTTP 요청을 도메인 객체로 변환하고, 도메인 결과를 JSON으로 직렬화한다"*
- **⓪ Frameworks**: *"Spring이 의존성 주입을 한다, JPA가 SQL을 생성한다"*

## Spring Boot 패키지 매핑

실무에서는 보통 다음과 같이 매핑된다:

| Uncle Bob의 원 | Spring Boot 패키지 | 예시 클래스 |
| --- | --- | --- |
| ③ Domain (Entities) | `domain/` | `Order`, `OrderStatus`, `Money`, `OrderPolicy` |
| ② Application (Use Cases) | `application/` | `PlaceOrderUseCase`, `OrderRepositoryPort`, `OrderService` |
| ① Interface Adapters | `adapter/` | `OrderController`, `OrderJpaAdapter`, `PaymentApiAdapter` |
| ⓪ Frameworks & Drivers | `infrastructure/` | `JpaConfig`, `WebClientConfig`, `SecurityConfig` |

### 패키지 구조 예시

```
com.example.order
│
├── domain/                          # ③
│   ├── order/
│   │   ├── Order.java
│   │   ├── OrderId.java
│   │   ├── OrderStatus.java
│   │   └── OrderPolicy.java
│   └── money/
│       └── Money.java
│
├── application/                     # ②
│   ├── port/
│   │   ├── in/
│   │   │   ├── PlaceOrderUseCase.java
│   │   │   └── PlaceOrderCommand.java
│   │   └── out/
│   │       ├── OrderRepositoryPort.java
│   │       └── PaymentGatewayPort.java
│   └── service/
│       └── OrderService.java
│
├── adapter/                         # ①
│   ├── in/
│   │   └── web/
│   │       ├── OrderController.java
│   │       └── PlaceOrderRequest.java
│   └── out/
│       ├── persistence/
│       │   ├── OrderJpaEntity.java
│       │   ├── OrderJpaRepository.java
│       │   └── OrderRepositoryAdapter.java
│       └── payment/
│           └── PaymentGatewayAdapter.java
│
├── infrastructure/                  # ⓪
│   └── config/
│       ├── JpaConfig.java
│       ├── WebClientConfig.java
│       └── SecurityConfig.java
│
└── OrderApplication.java
```

### 자주 헷갈리는 점 — JPA Entity는 어디?

`OrderJpaEntity`는 **`adapter/out/persistence/`** 에 위치한다 (① Interface Adapter). 이유:

- 도메인 객체(`Order`)와 JPA 엔티티(`OrderJpaEntity`)는 **별개의 클래스**
- JPA 엔티티는 *"도메인 ↔ DB 형식 번역"* 역할 → 어댑터에 속함
- `infrastructure/`는 **순수 설정 코드**(`@Configuration` 클래스들)만 들어감

```
[Application]
   │ OrderRepositoryPort.save(order)
   ↓
[Adapter ①]  ← OrderJpaEntity, OrderJpaRepository, OrderRepositoryAdapter
   │ JPA로 번역
   ↓
[Frameworks ⓪]  ← Hibernate, EntityManager (외부 라이브러리)
   ↓
[Database ⓪]
```

## 의존성 위반의 신호

도메인 패키지의 import 문만 봐도 위반 여부를 알 수 있다:

| 도메인에 보이면 안 되는 import | 의미 |
| --- | --- |
| `jakarta.persistence.*` | JPA가 도메인 침범 |
| `org.springframework.*` | Spring이 도메인 침범 |
| `com.fasterxml.jackson.*` | 직렬화 라이브러리가 도메인 침범 |
| `jakarta.servlet.*` | 웹 레이어가 도메인 침범 |
| `org.hibernate.*` | ORM 구현체가 도메인 침범 |

→ **코드 리뷰 시 도메인 패키지의 import 문 검사** 만으로 1차 검증 가능.

### 자동화

빌드 시 의존성 방향을 강제하는 도구:

- **ArchUnit** — Java 진영의 표준
- **Modulith** (Spring) — 모듈 간 의존성 검증

```java
// ArchUnit 예시
@Test
void domainShouldNotDependOnFramework() {
    noClasses().that().resideInAPackage("..domain..")
        .should().dependOnClassesThat()
        .resideInAnyPackage("..adapter..", "..infrastructure..");
}
```

## 명시적 의존 vs 암묵적 결합

Import 검사로 잡히는 위반은 **명시적 의존**뿐이다. 그러나 import 화살표가 올바른 방향이어도 **암묵적 결합**은 남을 수 있다. 이 차이가 Clean Architecture를 단순한 폴더 규칙이 아니라 **의미적 결합까지 보는 사고 방식**으로 이해하는 출발점이다.

### 예시 — Domain enum 공유 패턴

한국 실무에서 흔한 패턴:

```java
// domain/event/EventStatus.java
package com.example.event.domain;
public enum EventStatus { PENDING, SENT, FAILED, DEAD }
// ← 외부 import 0. Dependency Rule 준수.

// adapter/out/persistence/EventEntity.java
package com.example.event.adapter.out.persistence;

import com.example.event.domain.EventStatus;  // ← adapter → domain (올바른 방향)
import jakarta.persistence.*;

@Entity
public class EventEntity {
    @Enumerated(EnumType.STRING)
    private EventStatus status;
}
```

### Import만 보면 — 위반 없음

```
adapter (바깥) ──→ domain (안쪽)
                       ↑
                   ✅ 올바른 방향
```

ArchUnit으로 검증해도 통과한다. 도메인 enum은 외부 패키지를 import하지 않는다.

### 그런데 실제로는 — 묵시적 계약이 형성됨

`@Enumerated(EnumType.STRING)`이 붙는 순간 **enum 값의 이름(`"PENDING"`)이 DB 데이터 계약**이 된다.

```
DB의 status 컬럼 값:
    "PENDING", "SENT", ...
    └─ 이 문자열은 EventStatus.name() 결과
       → DB가 이 이름들에 묵시적으로 의존
```

도메인 enum 이름을 변경하면:

```java
// 도메인 리팩토링
public enum EventStatus {
    AWAITING_DISPATCH,  // PENDING에서 변경
    SENT, FAILED, DEAD
}
```

→ 컴파일은 됨. 그러나 DB의 기존 row는 여전히 `"PENDING"` 문자열. 앱 시작 시 `IllegalArgumentException` 발생, 모든 기존 데이터 읽기 실패.

**도메인 코드를 바꿨을 뿐인데 DB가 망가진다.**

### 두 차원의 비교

| 차원 | 정의 | 검증 방법 | 위 예시 |
| --- | --- | --- | --- |
| **명시적 의존 (Explicit)** | 코드가 다른 패키지를 import 함 | import 문 검사, ArchUnit | ✅ 위반 없음 |
| **암묵적 결합 (Implicit)** | 코드는 안 보이지만 의미적으로 묶임 | 변경 영향 분석 (사고 실험) | ⚠️ 결합 있음 |

### 그림으로

```
┌─────────────────────────┐         ┌──────────────────────────┐
│  domain/EventStatus     │         │  adapter/EventEntity      │
│                         │ ←─────  │  @Enumerated(STRING)      │
│  "import 0, 깨끗"        │  명시적  │  private EventStatus s;   │
└────────────┬────────────┘  의존    └───────────┬──────────────┘
             │                                    │
             │  암묵적 결합                         │
             │  (enum.name()이 DB 컨트랙트가 됨)    │
             ↓                                    ↓
        ┌──────────────────────────────────┐
        │   DB: status VARCHAR              │
        │   값: "PENDING", "SENT", ...       │
        └──────────────────────────────────┘
```

→ **import 화살표는 올바르지만, 의미적 화살표는 도메인을 묶고 있음.**

### Clean Architecture의 두 해석

| 해석 | 입장 | 결과 |
| --- | --- | --- |
| **엄격한 해석** | "암묵적 결합도 의존이다" | enum 분리 (Domain enum + JPA enum + Mapper) |
| **실용적 해석** | "명시적 의존만 막으면 된다" | enum 공유 (한국 실무 다수) |

둘 다 일관된 입장이다. **무엇을 "위반"으로 볼지의 정의가 다른 것**일 뿐.

### 진짜 분리는 어떻게 다른가

```java
// 도메인 enum (자유롭게 변경 가능)
public enum EventStatus { AWAITING_DISPATCH, SENT, FAILED, DEAD }

// JPA enum (DB 계약 보존)
public enum EventStatusJpa { PENDING, SENT, FAILED, DEAD }

// 매퍼가 흡수
class PersistenceMapper {
    static EventStatus toDomain(EventStatusJpa jpa) {
        return switch (jpa) {
            case PENDING -> EventStatus.AWAITING_DISPATCH;
            case SENT -> EventStatus.SENT;
            ...
        };
    }
}
```

→ DB 계약은 `EventStatusJpa`가 책임지고, 도메인 enum은 비즈니스 어휘로 자유. **명시적·암묵적 모두 분리.**

### 판단 기준

암묵적 결합을 감수할지 분리할지의 결정은 **변경 가능성 × 변경 비용**으로 판단:

| 상황 | 추천 |
| --- | --- |
| enum 이름이 안정적, DB row 적음 | 공유 (실용적 해석) |
| enum 이름 변경 가능성 있음, DB row 많음 | 분리 (엄격한 해석) |
| 학습 / 시연 / 장기 운영 시스템 | 분리 권장 |
| 프로토타입 / 단기 프로젝트 | 공유 권장 |

### 핵심 직관

> **Dependency Rule은 의존성의 "방향"만 본다. 그러나 진짜 결합은 "방향이 아닌 묵시적 계약"에서도 생긴다.**
>
> Clean Architecture를 깊이 이해하려면 import 검사를 넘어 *"이 변경이 다른 곳에 어떤 영향을 주는가"* 를 사고 실험으로 추적해야 한다.

암묵적 결합이 발생하는 다른 흔한 패턴:

- **DTO와 Domain Entity 공유** — DTO 필드가 곧 API 계약, 도메인 변경이 외부 인터페이스 변경
- **Domain ID 타입을 DB AUTO_INCREMENT에 의존** — 도메인이 DB 시퀀스를 가정
- **Domain Exception이 HTTP Status에 매핑된다는 묵시적 약속** — 도메인 예외 종류를 바꾸면 API 응답 코드 변경

→ 이런 결합들도 **import는 깨끗하지만 변경이 전파되는** 케이스. Clean Architecture의 "정신"을 따르려면 함께 점검해야 한다.

## 의존성 분리가 가져오는 실질 이득

### 1. 단위 테스트가 빠르고 쉬움

```java
// 도메인 단위 테스트 — Spring Context 불필요
@Test
void order_cannotBeNegative() {
    assertThrows(IllegalArgumentException.class,
        () -> Order.create(Money.of(-1000)));
}
```

→ Spring Boot Test 5초 vs 도메인 단위 테스트 50ms. **테스트 속도 100배**.

### 2. 인프라 교체 시 도메인 무영향

| 변경 시나리오 | 도메인 영향 |
| --- | --- |
| JPA → jOOQ | 무영향 (어댑터만 교체) |
| PostgreSQL → MongoDB | 무영향 |
| REST → gRPC | 무영향 |
| Spring → Quarkus | 무영향 |

→ 도메인이 비즈니스를 표현하지 기술을 표현하지 않음.

### 3. 비즈니스 가독성 향상

```java
public class Order {
    public boolean canCancel(CancellationPolicy policy) {
        return status == PLACED
            && hoursElapsed() < policy.maxCancelHours();
    }

    public void cancel(String reason) {
        if (status == DELIVERED) {
            throw new IllegalStateException("배송 완료된 주문은 취소 불가");
        }
        this.status = CANCELLED;
        this.cancelReason = reason;
    }
}
```

→ 비개발자(기획·운영팀)에게도 *"이런 비즈니스 규칙이 있구나"* 가 읽힘.

## 함정 / 자주 하는 오해

### ① "Clean Architecture는 폴더 구조다"

**아니다.** 핵심은 **의존성 방향**. 폴더가 깔끔해도 의존성이 잘못 흐르면 위반.

### ② "모든 프로젝트에 Clean Architecture를 적용해야 한다"

**아니다.** 보일러플레이트 비용이 작지 않다. 다음 경우엔 과한 적용:

- 프로토타입 / MVP
- 단순 CRUD 위주 (도메인 로직이 거의 없음)
- 단기 프로젝트 (3개월 미만)
- 작은 팀 (1-2명)

다음 경우엔 가치 있음:

- 장기 운영 (3년 이상)
- 복잡한 비즈니스 규칙
- 여러 외부 시스템 연동
- 인프라 교체 가능성 있음

### ③ "도메인 객체가 JPA 어노테이션을 갖고 있어도 동작하니까 OK"

**동작은 한다.** 다만:

- DB 마이그레이션이 도메인 변경을 강제함
- 도메인 단위 테스트에 JPA 의존성 필요
- 다른 영속성으로 교체 불가능

→ 단기적으로는 비용 없음, 장기적으로 부채.

### ④ "Use Case와 Service는 같은 거 아닌가?"

**비슷하지만 강조점이 다름.**

- **Service** (전통): 데이터 조작 중심, 트랜잭션 경계
- **Use Case** (Clean Arch): 사용자 시나리오 1개 = 클래스 1개, 입출력 명령 객체로 명시적 표현

```java
// 전통 Service — 여러 메소드 묶음
class OrderService {
    public Order create(...) { ... }
    public Order cancel(...) { ... }
    public Order refund(...) { ... }
    public List<Order> search(...) { ... }
}

// Clean Architecture Use Case — 시나리오별 분리
class PlaceOrderUseCase { ... }
class CancelOrderUseCase { ... }
class RefundOrderUseCase { ... }
class SearchOrdersUseCase { ... }
```

→ Use Case 분리는 단일 책임 원칙(SRP)의 강한 적용.

### ⑤ "Domain Entity와 JPA Entity를 분리하면 매퍼 보일러플레이트가 너무 많다"

**맞다, 비용이 있다.** 완화 방법:

- **MapStruct** — 컴파일 타임 매퍼 자동 생성
- **공통 추상 매퍼** — 공통 필드는 베이스 클래스로
- **단순 케이스는 record + 정적 팩토리** — 매퍼 클래스 없이 변환

규모가 작으면 매퍼 부담이 더 클 수 있으므로, **분리의 비용/이득을 프로젝트 단위로 판단**.

## 사고 모델

> **"비즈니스 규칙이 프레임워크를 모를수록 좋다."**
>
> 도메인은 회사가 망해도 살아남는다. 프레임워크는 5년마다 바뀐다. 둘 중 무엇이 무엇을 의존해야 하는가?

### 핵심 직관 3가지

1. **Dependency Rule = 의존성은 안쪽으로만**
2. **Port는 안쪽에서 정의, Adapter는 바깥에서 구현** (의존성 역전)
3. **JPA 엔티티는 도메인이 아니라 어댑터** (번역 코드는 ①에 속함)

### 위반 여부 판단 한 줄 체크

> *"`domain/` 패키지에서 `jakarta.*`, `org.springframework.*` import가 보이면 위반."*

## 참고

- [Robert C. Martin — The Clean Architecture (2012)](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
- [Uncle Bob — Clean Architecture (책, 2017)](https://www.amazon.com/Clean-Architecture-Craftsmans-Software-Structure/dp/0134494164)
- [Hexagonal Architecture (Alistair Cockburn, 2005)](https://alistair.cockburn.us/hexagonal-architecture/) — 유사 개념의 원조
- [ArchUnit](https://www.archunit.org/) — Java 의존성 규칙 자동 검증
- [Spring Modulith](https://spring.io/projects/spring-modulith) — Spring 진영의 모듈 검증 도구
