---
title: 커서 기반 페이지네이션
type: pattern
tags: [spring, spring-data, jpa, pagination, keyset, offset, index, performance]
related:
  - "[[pageable]]"
last_reviewed: 2026-08-23
publish: true
---

# 커서 기반 페이지네이션

페이지 번호로 위치를 지정하는 방식은 두 곳에서 무너진다. 깊은 페이지에서 비용이 커지고, 조회 사이에 데이터가 바뀌면 경계가 밀린다. 커서 방식은 위치를 **번호가 아니라 값으로** 지정해 둘 다 피한다. 번호 페이징 쪽은 [[pageable]] 참조.

## 목차

- [개요](#개요)
- [시나리오 — 주문 100만 건을 최신순으로 순회한다](#시나리오--주문-100만-건을-최신순으로-순회한다)
  - [요구사항](#요구사항)
  - [❌ OFFSET으로 건너뛴다](#-offset으로-건너뛴다)
  - [✅ 마지막으로 본 값을 커서로 넘긴다](#-마지막으로-본-값을-커서로-넘긴다)
  - [무엇이 좋아졌나](#무엇이-좋아졌나)
- [커서 조건은 펼쳐 쓴다](#커서-조건은-펼쳐-쓴다)
- [왜 중복이 생기지 않나](#왜-중복이-생기지-않나)
- [인덱스가 없으면 의미가 없다](#인덱스가-없으면-의미가-없다)
- [대가](#대가)
- [Spring Data의 지원](#spring-data의-지원)
- [선택 기준](#선택-기준)
- [관련 노트](#관련-노트)

## 개요

`OFFSET`은 "앞에서 몇 번째"라는 **상대 위치**다. 그 앞의 행이 몇 개인지에 의존하므로 두 가지 성질을 갖는다.

- DB가 건너뛸 행도 읽어야 한다 → 페이지가 깊어질수록 느려진다
- 앞쪽에 행이 추가되면 같은 번호가 다른 행을 가리킨다 → 이미 본 행이 다시 나온다

커서 방식(keyset pagination)은 위치를 "이 값 다음"이라는 **절대 위치**로 잡는다. 마지막으로 본 행의 정렬 키를 그대로 다음 요청의 시작점으로 넘긴다. 앞에 몇 개가 있는지 세지 않으므로 두 성질이 함께 사라진다.

## 시나리오 — 주문 100만 건을 최신순으로 순회한다

### 요구사항

- 주문 전체를 최신순으로 훑어 외부 시스템에 내보낸다
- 한 번에 10건씩 가져온다
- 순회하는 동안에도 새 주문이 계속 들어온다

### ❌ OFFSET으로 건너뛴다

페이지 번호를 1씩 늘려 가며 끝까지 넘긴다.

```java
int page = 0;
while (true) {
    List<Order> batch = repo.findByStatus(PAID,
            PageRequest.of(page++, 10, Sort.by("createdAt").descending().and(Sort.by("id").descending())));
    if (batch.isEmpty()) break;
    export(batch);
}
```

```sql
-- 10,001번째 요청: 100,010행을 읽고 10행만 남긴다
select * from orders where status = 'PAID'
order by created_at desc, id desc
limit 10 offset 100000;
```

문제점:

- **깊어질수록 느려진다.** `OFFSET 100000`은 DB가 10만 행을 읽어서 버린 뒤 그다음부터 준다는 뜻이다. 첫 요청이 10행을 읽는 동안 10,001번째 요청은 100,010행을 읽는다.
- **전체 순회의 총비용이 제곱으로 늘어난다.** 10만 페이지를 다 넘기면 읽는 행 수의 합이 `10 + 20 + ... + 1,000,000`이다. 100만 건을 내보내려고 5,000억 행을 읽는다.
- **순회 중 삽입이 경계를 밀어낸다.** 새 주문이 앞쪽(최신)에 들어오면 `OFFSET 10`이 가리키는 행이 한 칸 밀린다. 이미 내보낸 주문이 다음 배치에 다시 나오고, 그만큼 어떤 주문은 어느 배치에도 나오지 않는다.

### ✅ 마지막으로 본 값을 커서로 넘긴다

번호 대신 마지막 행의 `(createdAt, id)`를 다음 요청의 시작점으로 쓴다.

```java
LocalDateTime cursorCreatedAt = null;
Long cursorId = null;

while (true) {
    List<Order> batch = (cursorCreatedAt == null)
            ? repo.findFirstPage(PAID, PageRequest.of(0, 10))
            : repo.findNextPage(PAID, cursorCreatedAt, cursorId, PageRequest.of(0, 10));
    if (batch.isEmpty()) break;

    export(batch);

    Order last = batch.get(batch.size() - 1);   // 이번 배치의 마지막 행이 다음 커서
    cursorCreatedAt = last.getCreatedAt();
    cursorId = last.getId();
}
```

```sql
select * from orders
where status = 'PAID'
  and (created_at, id) < ('2026-08-22 10:00:00', 8842)   -- 커서
order by created_at desc, id desc
limit 10;
```

커서 조건이 `WHERE`이므로 **이미 본 행들이 결과 집합에서 아예 빠진다.** 정렬과 같은 방향으로 비교하기 때문에(`created_at DESC`로 정렬하면서 "이 시각보다 이전"을 찾는다) 인덱스에서 그 지점을 찾아 10행만 읽고 멈춘다. 몇 번째 요청이든 읽는 행은 10행이다.

`PageRequest.of(0, 10)`을 넘기는 이유는 개수만 지정하기 위해서다. 페이지 번호가 항상 0이므로 `OFFSET 0`이고, 실제 위치는 `WHERE`의 커서 조건이 정한다.

다음 배치가 있는지 미리 알아야 하면 `PageRequest.of(0, 11)`로 하나 더 요청한다. 11건이 오면 다음이 있고, 10건만 담아 반환한다. `Slice`가 `size + 1`을 조회하는 것과 같은 방법이다.

### 무엇이 좋아졌나

| | ❌ `OFFSET` | ✅ 커서 |
|---|---|---|
| 10,001번째 요청이 읽는 행 | 100,010 | 10 |
| 100만 건 전체 순회의 총 읽기 | 약 5,000억 행 | 100만 행 |
| 순회 중 앞쪽에 삽입되면 | 경계가 밀려 중복·누락 | 영향 없음 |
| 임의 페이지로 이동 | 가능 | 불가 |

## 커서 조건은 펼쳐 쓴다

위 SQL의 `(created_at, id) < (?, ?)` 같은 행 값 비교는 DB와 방언에 따라 지원 여부가 갈린다. 조건을 펼쳐 쓰면 그 차이에 걸리지 않는다.

```java
@Query("select o from Order o where o.status = :status "
     + "and (o.createdAt < :cursorCreatedAt "
     + "  or (o.createdAt = :cursorCreatedAt and o.id < :cursorId)) "
     + "order by o.createdAt desc, o.id desc")
List<Order> findNextPage(@Param("status") OrderStatus status,
                         @Param("cursorCreatedAt") LocalDateTime cursorCreatedAt,
                         @Param("cursorId") Long cursorId,
                         Pageable pageable);
```

`or` 절이 `created_at`이 같은 행들을 `id`로 가르는 부분이다. 이걸 빼고 `o.createdAt < :cursorCreatedAt`만 쓰면, 커서와 `created_at`이 같은 행 전체가 한꺼번에 건너뛰어진다. 같은 시각에 만들어진 주문이 15건이면 그중 아직 안 본 것까지 사라진다.

정렬 키가 셋이면 조건도 한 단 더 깊어진다. 그래서 커서에 쓰는 정렬 키는 둘(정렬 기준 + `id`)로 유지하는 편이 낫다.

## 왜 중복이 생기지 않나

순회 사이에 새 주문이 하나 들어왔다고 하자.

- **`OFFSET` 방식** — `OFFSET 10`은 여전히 "앞에서 11번째"다. 앞쪽에 행이 하나 끼어들면 경계가 한 칸 밀려서, 1페이지의 마지막 주문이 2페이지의 첫 주문으로 다시 나온다.
- **커서 방식** — 시작점이 "`(2026-08-22 10:00:00, 8842)`보다 이전"으로 고정되어 있다. 앞쪽에 몇 건이 추가되든 그 경계는 움직이지 않는다.

[[pageable]]의 tie-breaker 이야기와 같은 원리의 연장이다. 정렬이 **전순서**여야 커서가 한 지점을 정확히 가리킬 수 있다. 그래서 커서에 쓰는 키에는 반드시 유일한 값(`id`)이 포함되어야 한다. `created_at`만으로 커서를 만들면 같은 시각의 행들 사이에서 경계가 정해지지 않아, 커서로 바꾼 의미가 없어진다.

## 인덱스가 없으면 의미가 없다

커서 방식의 이득은 인덱스를 타고 시작 지점으로 바로 간다는 데서 온다. 정렬 키 순서대로 복합 인덱스가 있어야 한다.

```sql
create index idx_orders_status_created_id on orders (status, created_at desc, id desc);
```

인덱스가 없으면 `WHERE` 조건이 있어도 전체를 스캔해 걸러내므로 `OFFSET`과 다를 바가 없어진다. 커서로 바꾸고도 느리면 실행 계획을 먼저 본다.

인덱스 컬럼 순서는 `WHERE`의 등치 조건(`status`) → 정렬 키(`created_at`, `id`) 순이다. 정렬 방향까지 맞추면 DB가 인덱스를 역주행하지 않아도 된다.

## 대가

- **임의의 페이지로 건너뛸 수 없다.** "7페이지로 이동"이 불가능하다. 이전/다음만 가능하다.
- **전체 페이지 수를 알 수 없다.** 필요하면 count 쿼리를 따로 던져야 한다.
- **정렬 키에 `null`이 있으면 안 된다.** `null`을 비교 연산자로 다루는 방식이 DB마다 달라 커서 경계가 어긋난다.
- **정렬을 클라이언트가 자유롭게 바꾸기 어렵다.** 정렬이 바뀌면 커서의 의미도 바뀌므로 기존 커서를 버려야 한다.
- **커서를 클라이언트에게 넘겨야 한다.** `createdAt`·`id`를 그대로 노출하기 싫으면 인코딩해 불투명한 문자열로 준다. 다만 인코딩은 가리는 것일 뿐 검증이 아니다. 커서를 조작해 보내도 다른 사용자의 데이터가 보이지 않도록, 소유자·상태 조건은 커서와 별개로 항상 `WHERE`에 넣는다.

## Spring Data의 지원

Spring Data 3.1부터 이 방식을 `ScrollPosition`과 `Window<T>`로 지원한다. 커서 조건을 직접 쓰지 않고 `ScrollPosition.keyset()`을 넘기면 위 `where` 절을 만들어 준다. 개수는 `Pageable`이 아니라 메서드 이름(`findFirst10By...`)으로 지정한다.

```java
interface OrderRepository extends Repository<Order, Long> {
    Window<Order> findFirst10ByStatusOrderByCreatedAtDesc(OrderStatus status, ScrollPosition position);
}
```

`WindowIterator`를 쓰면 커서를 직접 들고 다니지 않아도 된다.

```java
WindowIterator<Order> orders = WindowIterator
        .of(position -> repo.findFirst10ByStatusOrderByCreatedAtDesc(PAID, position))
        .startingAt(ScrollPosition.keyset());          // 첫 요청은 커서 없이

while (orders.hasNext()) {
    export(orders.next());
}
```

직접 순회하려면 `Window<T>`에서 마지막 원소의 위치를 꺼내 다음 요청에 넘긴다. `Slice`처럼 `hasNext()`·`isLast()`를 제공한다.

```java
Window<Order> window = repo.findFirst10ByStatusOrderByCreatedAtDesc(PAID, ScrollPosition.keyset());
while (!window.isEmpty()) {
    window.forEach(this::export);
    if (window.isLast()) break;
    window = repo.findFirst10ByStatusOrderByCreatedAtDesc(PAID, window.positionAt(window.size() - 1));
}
```

직접 쓸 때와 달라지는 점이 셋 있다.

- **tie-breaker를 붙여 준다.** Spring Data가 정렬 순서에 기본 키를 덧붙여 결과가 유일해지도록 보정한다. 위 메서드 이름에 `IdDesc`를 적지 않아도 된다.
- **정렬 키는 `null`이 아니어야 한다.** 비교 연산자의 `null` 처리가 저장소마다 달라서, nullable 프로퍼티를 정렬 키로 쓰면 결과가 예상과 달라진다.
- **정렬 키가 결과에 포함되어야 한다.** 반환 타입에 매핑되지 않은 프로퍼티로는 정렬할 수 없다. 다음 커서를 만들 값이 결과 안에 있어야 하기 때문이다.

## 선택 기준

- **사용자가 페이지 번호를 누르는 화면이면 번호 페이징** — 커서로는 "7페이지로 이동"을 만들 수 없다.
- **무한 스크롤·배치 순회·데이터 내보내기면 커서** — 깊이와 무관하게 비용이 일정하고, 순회 중 삽입에 흔들리지 않는다.
- **커서 키에는 반드시 유일한 값을 포함한다** — 없으면 커서가 한 지점을 가리키지 못한다.
- **복합 인덱스를 먼저 만든다** — 인덱스 없는 커서는 `OFFSET`과 같다.
- **커서와 별개로 권한 조건을 항상 넣는다** — 커서는 위치 정보일 뿐 접근 제어가 아니다.

## 관련 노트

- [[pageable]] — 번호 기반 페이징. `Page`/`Slice` 선택, 정렬 tie-breaker, 컨트롤러 바인딩
