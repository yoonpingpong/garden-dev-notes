---
title: Pageable과 페이징 조회
type: concept
tags: [spring, spring-data, jpa, pagination, pageable, sort]
related:
  - "[[keyset-pagination]]"
  - "[[persistence-state-and-flush]]"
last_reviewed: 2026-08-23
publish: true
---

# Pageable과 페이징 조회

`Pageable`은 "몇 번째 페이지를, 몇 개씩, 어떤 순서로"를 담은 값이다. 리포지토리 메서드에 넘기면 Spring Data가 이 값을 `LIMIT`/`OFFSET`과 `ORDER BY`로 번역한다.

## 목차

- [개요](#개요)
- [Pageable을 만드는 법](#pageable을-만드는-법)
- [반환 타입 셋 — Page, Slice, List](#반환-타입-셋--page-slice-list)
- [정렬](#정렬)
- [웹 요청에서 받을 때](#웹-요청에서-받을-때)
  - [기본값과 상한은 다른 설정이다](#기본값과-상한은-다른-설정이다)
  - [sort는 여러 번 줄 수 있다](#sort는-여러-번-줄-수-있다)
  - [one-indexed-parameters의 비대칭](#one-indexed-parameters의-비대칭)
- [Page를 그대로 응답으로 내보내지 않는다](#page를-그대로-응답으로-내보내지-않는다)
- [컬렉션 fetch join과 함께 쓰면 메모리에서 자른다](#컬렉션-fetch-join과-함께-쓰면-메모리에서-자른다)
- [OFFSET의 한계](#offset의-한계)
- [관련 노트](#관련-노트)

## 개요

`Pageable`은 인터페이스이고, 실제로 쓰는 구현체는 `PageRequest`다. 담는 정보는 셋이다.

- **페이지 번호** — `0`부터 시작한다
- **페이지 크기** — 한 페이지에 담을 개수
- **정렬** — `Sort`

```java
public interface OrderRepository extends JpaRepository<Order, Long> {
    Page<Order> findByStatus(OrderStatus status, Pageable pageable);
}
```

**페이지 번호가 0부터라는 것이 첫 번째 함정이다.** 사용자에게 보이는 "1페이지"는 `page=0`이다. 화면과 API 사이에서 이 변환을 어디서 할지 정해두지 않으면 첫 페이지가 빠지거나 두 번 나온다.

## Pageable을 만드는 법

```java
PageRequest.of(0, 20);                                     // 첫 페이지, 20개
PageRequest.of(2, 20, Sort.by("createdAt").descending());  // 세 번째 페이지, 최신순
Pageable.ofSize(20);                                       // 페이지 번호 0, 크기 20
Pageable.unpaged();                                        // 페이징하지 않음
Pageable.unpaged(Sort.by("createdAt"));                    // 페이징 없이 정렬만
```

`Pageable.unpaged()`는 페이징을 끄는 값이다. "조건에 따라 전체 조회"라는 분기를 `if (pageable == null)` 같은 검사 없이 표현할 수 있다. 다만 전체 조회가 되므로 데이터가 커지면 그 분기 자체가 위험해진다.

한 메서드에 `Pageable`과 `Sort`를 **같이 선언할 수 없다.** `Pageable`이 이미 `Sort`를 갖고 있어서 정렬 정보의 출처가 둘이 되기 때문이다. 정렬만 필요하면 `Sort`만, 페이징이 필요하면 `Pageable`만 받는다.

## 반환 타입 셋 — Page, Slice, List

같은 `Pageable`을 넘겨도 반환 타입에 따라 얻는 정보와 나가는 쿼리가 다르다.

| 반환 타입 | 알 수 있는 것 | count 쿼리 |
|---|---|---|
| `Page<T>` | 전체 개수, 전체 페이지 수, 다음 페이지 유무 | 나간다 |
| `Slice<T>` | 다음 `Slice`가 있는지만 | 안 나간다 |
| `List<T>` | 없음 | 안 나간다 |

`Page`가 전체 개수를 아는 방법은 count 쿼리를 한 번 더 던지는 것이다. 저장소에 따라 이 비용이 크므로, 큰 결과를 훑기만 할 때는 `Slice`로 충분하다.

`List<T>`로 받아도 **`Pageable`은 그대로 적용된다.** 조회 범위를 그 페이지로 제한하는 것은 같고, `Page`를 만들기 위한 메타데이터와 그에 필요한 count 쿼리만 생기지 않는다. 이 페이지의 데이터만 쓰고 다음 페이지 유무조차 화면에서 쓰지 않는다면 이것이 가장 가벼운 선택이다.

## 정렬

`Sort`의 인자는 **엔티티의 프로퍼티 이름**이다. DB 컬럼명이 아니다.

```java
Sort.by("createdAt")                                    // 엔티티 필드
Sort.by("member.name")                                  // 연관 관계를 점으로 탄다
Sort.by("createdAt").descending().and(Sort.by("id"))    // 정렬 키 둘
```

`Sort`는 정렬 키의 **순서 있는 목록**이다. 첫 키로 정렬하고, 값이 같은 행끼리는 다음 키로 가른다.

**정렬 키가 유일하지 않으면 페이징이 성립하지 않는다.** `created_at`이 같은 행이 여러 개 있으면 그들 사이의 순서는 SQL 수준에서 정해지지 않는다. `ORDER BY created_at DESC LIMIT 10 OFFSET 10`은 두 번째 요청에서 앞의 10개를 다시 정렬하는데, 그 결과가 첫 요청과 같을 보장이 없다. 어떤 행은 두 페이지에 걸쳐 나오고 그만큼 다른 행은 어느 페이지에도 나오지 않는다.

그래서 정렬 마지막에 유일한 값을 붙여 **전순서**로 만든다.

```java
Sort.by("createdAt").descending()
    .and(Sort.by("id").descending());     // 동순위를 가르는 키
```

데이터가 적을 때는 대개 우연히 일치하고, 운영에서 데이터가 쌓인 뒤에 어긋난다. 로컬에서 재현되지 않는 종류의 버그다.

## 웹 요청에서 받을 때

컨트롤러 파라미터에 `Pageable`을 선언하면 Spring MVC가 요청 파라미터에서 바인딩한다. 파라미터 이름은 `page`, `size`, `sort`다.

```
GET /orders?page=2&size=50&sort=createdAt,desc&sort=id,desc
```

### 기본값과 상한은 다른 설정이다

`size`에 얽힌 숫자가 두 개 나오는데 서로 다른 것이다. 하나는 **값이 없을 때 채우는 기본값**이고, 다른 하나는 **값이 있을 때 걸리는 상한**이다.

| 설정 | 기본값 | 언제 쓰이나 |
|---|---|---|
| `spring.data.web.pageable.default-page-size` | 20 | 요청에 `size`가 **없을 때** 그 값으로 채운다 |
| `spring.data.web.pageable.max-page-size` | 2000 | 요청의 `size`가 이 값을 **넘을 때** 여기까지 잘라낸다 |

```
GET /orders                → size=20     (기본값이 채워진다)
GET /orders?size=100       → size=100    (상한 이하이므로 그대로)
GET /orders?size=5000      → size=2000   (상한으로 잘린다)
GET /orders?size=0         → size=20     (1보다 작으면 예외가 아니라 기본값)
```

`size`가 1보다 작으면 예외를 던지지 않고 기본값으로 되돌린다. 즉 잘못된 `size`는 요청을 실패시키지 않고 조용히 다른 값이 된다.

상한을 낮추면(`max-page-size=100`) 클라이언트가 아무리 큰 값을 보내도 100을 넘지 못한다. 외부에 열린 API라면 상한을 기본값 2000보다 낮추는 편이 안전하다. 한 번의 요청으로 2000건을 조회하는 것을 허용할 이유가 대개 없다.

### sort는 여러 번 줄 수 있다

`sort` 파라미터를 여러 번 주면 등장한 순서가 그대로 정렬 우선순위가 된다.

```
GET /orders?sort=createdAt,desc&sort=id,desc
→ ORDER BY created_at DESC, id DESC
```

앞 절에서 손으로 붙였던 tie-breaker를 URL로 표현한 것이 이것이다.

한 파라미터 안에 콤마로 여러 프로퍼티를 묶을 수도 있고, 이때는 방향이 공유된다.

```
sort=createdAt,desc            → created_at DESC
sort=createdAt,id,desc         → created_at DESC, id DESC   (마지막 토큰이 방향)
sort=createdAt,desc&sort=id    → created_at DESC, id ASC    (방향을 생략하면 ASC)
```

**한 `sort` 파라미터 안에서는 방향을 하나만 줄 수 있다.** 파서가 방향으로 해석하는 것은 맨 마지막 토큰뿐이고, 그 앞의 토큰은 전부 프로퍼티 이름으로 본다.

```
sort=createdAt,asc,id,desc
→ 프로퍼티 [createdAt, asc, id]를 모두 DESC로 해석한다 (`asc`도 프로퍼티로 본다)
```

방향을 섞어야 하면 파라미터를 나눈다.

```
sort=createdAt,asc&sort=id,desc   → created_at ASC, id DESC
```

파서는 마지막 토큰을 먼저 `ignorecase`로, 그다음 방향으로 해석한다. 그래서 대소문자를 무시한 정렬은 **방향 뒤에** 붙인다.

```
sort=name,asc,ignorecase   → ORDER BY lower(name) ASC
sort=name,ignorecase,asc   → ❌ `ignorecase`가 프로퍼티로 해석된다
```

이 토큰은 해당 `Sort.Order`에 `ignoreCase` 플래그를 켜고, Spring Data JPA가 그 프로퍼티를 `lower()`로 감싼다. 쓰기 전에 볼 것이 셋이다.

- **String 프로퍼티에만 쓸 수 있다.** 숫자·날짜 프로퍼티에 걸면 `InvalidDataAccessApiUsageException`이 난다 — `Unable to ignore case of <타입> types, the property '<이름>' must reference a String`.
- **인덱스를 무력화한다.** `lower(name)`으로 정렬하면 `name`의 일반 인덱스를 타지 못한다. PostgreSQL에서는 함수 인덱스가 따로 필요하다.
- **MySQL에서는 대개 필요 없다.** 기본 콜레이션이 이미 대소문자를 구분하지 않으므로, `lower()`를 씌우면 인덱스만 잃고 얻는 것이 없다.

정렬 키를 클라이언트가 정하면 엔티티의 프로퍼티 이름이 그대로 URL에 노출된다. 외부에 열린 API라면 허용할 정렬 키를 서버가 화이트리스트로 두는 편이 낫다.

### one-indexed-parameters의 비대칭

`spring.data.web.pageable.one-indexed-parameters=true`를 켜면 요청의 `page=1`이 첫 페이지가 된다. 이 설정이 하는 일은 요청 파라미터를 파싱할 때 값에서 1을 빼는 것뿐이다. 그래서 **요청 해석만 1부터로 바뀌고, 응답으로 나가는 `Page`의 `number`는 여전히 0부터다.**

이 비대칭을 모르고 켜면 프론트가 받은 `number`에 다시 1을 더하게 되고, 어디서 보정하는지가 코드에 흩어진다. 응답 DTO를 직접 만들어 그 안에서 한 번만 변환하는 편이 낫다.

## Page를 그대로 응답으로 내보내지 않는다

`Page<T>`를 컨트롤러에서 그대로 반환하면 구현체인 `PageImpl`이 직렬화된다. 그 JSON 구조는 프레임워크 내부 구현이라 안정성이 보장되지 않고, Spring Data가 경고를 남긴다.

```
Serializing PageImpl instances as-is is not supported, meaning that there is no
guarantee about the stability of the resulting JSON structure!
```

Spring Data 3.3부터 제공하는 `PagedModel`을 쓰면 Spring HATEOAS를 끌어오지 않고도 안정적인 구조를 얻는다.

```java
return new PagedModel<>(page);
```

메서드마다 감싸는 대신 전역으로 적용하려면 `@EnableSpringDataWebSupport(pageSerializationMode = VIA_DTO)`를 켠다. 그러면 `PageImpl`을 반환해도 `PagedModel` 구조로 직렬화된다.

엔티티를 그대로 직렬화하면 노출하지 않으려던 필드가 함께 나가므로, 어느 방법을 쓰든 내용은 응답 전용 타입으로 변환한 뒤 담는다.

## 컬렉션 fetch join과 함께 쓰면 메모리에서 자른다

`Pageable`과 컬렉션 fetch join을 같이 쓰면 DB에서 페이징되지 않는다.

```java
@Query("select o from Order o join fetch o.items where o.status = :status")
Page<Order> findWithItems(@Param("status") OrderStatus status, Pageable pageable);
```

주문 1건에 상품 3건이 딸려 있으면 조인 결과는 3행이 된다. `LIMIT 10`은 주문 10건이 아니라 **행 10개**를 자르므로 마지막 주문의 상품이 누락된다. 그래서 Hibernate는 `LIMIT`을 붙이지 않고 전체를 읽어 메모리에서 자르며, 로그에 이 경고를 남긴다.

```
WARN  HHH90003004: firstResult/maxResults specified with collection fetch; applying in memory
```

로그 코드는 버전에 따라 다르다. Hibernate 6은 `HHH90003004`, Hibernate 5는 `HHH000104`이고 메시지 본문은 같다. 이 경고가 보이면 그 쿼리는 페이징되지 않은 것이고, 주문이 10만 건이면 10만 건과 그 상품 전체가 힙에 올라온다.

해결은 조회를 두 단계로 나누는 것이다. 페이징으로 ID만 얻고, 그 ID로 컬렉션을 가져온다.

```java
Page<Long> ids = repo.findIdsByStatus(PAID, pageable);            // ① 페이징은 ID만
List<Order> orders = repo.findWithItemsByIdIn(ids.getContent());  // ② fetch join은 페이징 없이
return new PageImpl<>(orders, pageable, ids.getTotalElements());
```

①은 컬렉션을 조인하지 않으므로 한 주문이 한 행이고, `LIMIT`이 의도한 대로 주문을 자른다. ②는 대상이 이미 확정되어 `LIMIT`이 필요 없으므로 fetch join을 해도 안전하다. 즉 페이징은 ①에서 DB가 한다.

②에서 놓치기 쉬운 것이 있다. **`IN` 조회 결과의 행 순서는 보장되지 않으므로 ①의 정렬을 ②에도 다시 적어야 한다.**

```java
@Query("select o from Order o join fetch o.items "
     + "where o.id in :ids order by o.createdAt desc, o.id desc")
List<Order> findWithItemsByIdIn(@Param("ids") List<Long> ids);
```

빠뜨리면 페이지 경계는 맞는데 페이지 안의 순서가 뒤섞인다. 정렬이 요청마다 바뀌어 쿼리에 박을 수 없으면, ①이 준 ID 순서대로 메모리에서 다시 세운다.

**`ToOne` 연관(`@ManyToOne`, `@OneToOne`)은 이 문제가 없다.** 행 수가 늘지 않으므로 fetch join과 페이징을 함께 써도 된다. 문제는 컬렉션(`@OneToMany`, `@ManyToMany`)뿐이다.

## OFFSET의 한계

`Pageable`의 페이지 번호는 `OFFSET`으로 번역된다. `OFFSET`은 "앞에서 몇 번째"라는 상대 위치이므로 두 가지 성질을 함께 갖는다.

- **건너뛸 행도 읽는다.** `OFFSET 100000`은 10만 행을 읽어 버린 뒤 그다음부터 준다. 페이지가 깊어질수록 느려지고, 전체를 순회하면 읽는 행 수가 제곱으로 늘어난다.
- **앞쪽에 행이 추가되면 경계가 밀린다.** 같은 페이지 번호가 다른 행을 가리키게 되어, 이미 본 행이 다음 페이지에 다시 나온다.

사용자가 페이지 번호를 누르는 목록 화면이라면 이 성질을 감수하고 그대로 쓴다. 깊은 페이지를 실제로 넘기거나 전체를 훑는 경우 — 관리자 목록, 배치 순회, 데이터 내보내기 — 에는 [[keyset-pagination]]의 커서 방식으로 바꾼다.

## 관련 노트

- [[keyset-pagination]] — `OFFSET`의 한계를 피하는 커서 방식
- [[persistence-state-and-flush]] — 같은 트랜잭션에서 엔티티를 수정한 뒤 페이징 조회를 하면, 조회 쿼리 실행 직전 flush가 일어나 수정 내용이 정렬·필터 결과에 반영된다

%%
확인 범위 — 이 노트의 프레임워크 동작 서술은 Spring Data 소스와 공식 문서로 확인한 것만 담았다.
확인하지 못해 뺀 항목은 daily/2026-08-23.md에 있다.
%%
