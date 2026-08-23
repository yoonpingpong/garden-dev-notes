---
title: 영속 상태와 flush
type: concept
tags: [spring, jpa, hibernate, transaction, persistence-context, dirty-checking]
related:
  - "[[transaction-proxy-boundary]]"
  - "[[pageable]]"
last_reviewed: 2026-08-17
publish: true
---

# 영속 상태와 flush

엔티티를 수정했는데 DB에 반영되지 않는 상황에는 원인이 둘 있고, 증상은 똑같아서 결과만 보고는 구분되지 않는다.

## 목차

- [한눈에](#한눈에)
- [개요](#개요)
- [시나리오 — 회원 이름 변경](#시나리오--회원-이름-변경)
  - [요구사항](#요구사항)
  - [❌ 트랜잭션이 없을 때](#-트랜잭션이-없을-때)
  - [❌ readOnly 트랜잭션일 때](#-readonly-트랜잭션일-때)
  - [✅ 쓰기 트랜잭션일 때](#-쓰기-트랜잭션일-때)
  - [세 경우 비교](#세-경우-비교)
- [flush는 언제 일어나나](#flush는-언제-일어나나)
- [상태를 직접 확인하기](#상태를-직접-확인하기)
- [로그로 구분하기](#로그로-구분하기)
- [준영속인데 반영되는 경우](#준영속인데-반영되는-경우)

## 한눈에

- 변경이 DB로 나가려면 두 조건이 모두 참이어야 한다. 엔티티가 영속 상태일 것, 그리고 flush가 일어날 것.
- 트랜잭션이 없으면 조회 직후 엔티티가 준영속이 되어 변경을 감지할 주체가 사라진다.
- readOnly 트랜잭션에서는 엔티티가 영속 상태로 남지만 flush가 막혀 UPDATE가 나가지 않는다.
- 두 경우 모두 결과는 "반영 안 됨"으로 같다. 원인이 다르므로 고치는 곳도 다르다. entityManager.contains로 상태를 직접 확인하면 갈린다.

## 개요

JPA에서 엔티티를 수정하는 코드는 대개 이렇게 생겼다. 조회하고, 값을 바꾸고, 끝이다. 저장 코드가 없다.

```java
Member member = memberRepository.findById(id).orElseThrow();
member.changeName("홍길동");
```

이것이 동작하는 이유는 변경 감지 때문이다. 영속성 컨텍스트는 관리 중인 엔티티의 최초 상태를 스냅샷으로 들고 있다가, flush 시점에 현재 값과 비교해 달라진 것이 있으면 UPDATE를 만든다.

여기에 조건이 둘 있다. 엔티티가 영속성 컨텍스트의 관리 대상이어야 하고, flush가 실제로 일어나야 한다. 관리 대상인지를 상태, flush가 일어나는지를 동작이라고 부르면 두 축은 서로 독립이다.

| 축 | 무엇이 결정하는가 | 무엇으로 확인하는가 |
|---|---|---|
| 상태 — 영속인가 준영속인가 | 트랜잭션의 유무 | entityManager.contains |
| 동작 — flush가 일어나는가 | readOnly 여부 | UPDATE 쿼리 |

둘 중 하나만 거짓이어도 결과는 똑같이 "값이 안 바뀐다"이다. 그래서 결과만 보고 원인을 추측하면 엉뚱한 곳을 고치게 된다.

## 시나리오 — 회원 이름 변경

### 요구사항

회원 한 명의 이름을 바꾼다. 세 버전의 메서드 본문은 완전히 같고 트랜잭션 설정만 다르다.

```java
Member member = memberRepository.findById(id).orElseThrow();
member.changeName("홍길동");
```

### ❌ 트랜잭션이 없을 때

```java
public void rename(Long id) {
    Member member = memberRepository.findById(id).orElseThrow();
    member.changeName("홍길동");
}
```

UPDATE가 나가지 않는다.

findById는 트랜잭션이 없어도 동작한다. Spring Data가 리포지토리 메서드에 트랜잭션을 걸어 두었기 때문이다. 문제는 그 트랜잭션이 조회를 마치는 즉시 끝난다는 것이다. 영속성 컨텍스트가 함께 닫히고 반환된 엔티티는 준영속이 된다.

그 다음 줄의 changeName은 아무도 추적하지 않는 자바 객체의 필드를 바꾼 것이다. 스냅샷과 비교할 주체가 없다.

### ❌ readOnly 트랜잭션일 때

```java
@Transactional(readOnly = true)
public void rename(Long id) {
    Member member = memberRepository.findById(id).orElseThrow();
    member.changeName("홍길동");
}
```

역시 UPDATE가 나가지 않는다. 다만 이유가 다르다.

이번에는 트랜잭션이 있으므로 영속성 컨텍스트가 메서드 끝까지 열려 있고, 엔티티는 영속 상태다. 변경도 컨텍스트가 알고 있다. 그런데 readOnly가 flush를 막는다. 하이버네이트의 FlushMode가 MANUAL로 설정되어 커밋 시점에 flush가 일어나지 않는다.

조회 메서드에 readOnly를 붙이라는 권고에는 성능 외에 이 안전장치 역할도 있다. 실수로 엔티티를 건드려도 DB가 오염되지 않는다.

### ✅ 쓰기 트랜잭션일 때

```java
@Transactional
public void rename(Long id) {
    Member member = memberRepository.findById(id).orElseThrow();
    member.changeName("홍길동");
}
```

UPDATE가 한 번 나간다.

엔티티가 영속 상태이고 flush도 막히지 않았다. 커밋 직전에 스냅샷과 현재 값이 비교되고 달라진 필드에 대해 UPDATE가 만들어진다.

UPDATE가 나가는 시점은 changeName을 호출한 직후가 아니라 커밋 시점이다. 그래서 같은 엔티티의 필드를 열 번 바꿔도 UPDATE는 한 번이다.

### 세 경우 비교

| | 트랜잭션 | 엔티티 상태 | flush | DB 반영 |
|---|---|---|---|---|
| 트랜잭션 없음 | 없음 | 준영속 | 대상 없음 | 안 됨 |
| readOnly | 있음 | 영속 | 막힘 | 안 됨 |
| 쓰기 | 있음 | 영속 | 일어남 | 됨 |

트랜잭션 없음과 readOnly는 마지막 칸이 같고 나머지가 다르다.

## flush는 언제 일어나나

flush를 직접 호출하는 코드는 보통 없다. 그래도 일어나는 시점이 셋 있다.

- 트랜잭션을 커밋하기 직전. 가장 흔한 경우다
- JPQL이나 Criteria로 만들어진 JPA 쿼리를 실행하기 직전. 변경 내용이 아직 DB에 없으면 그 쿼리가 잘못된 결과를 돌려주므로 하이버네이트가 먼저 내보낸다
- entityManager.flush()를 직접 호출할 때

JPA 쿼리를 보내기 전에 flush하는 이유는 조건을 평가하는 주체가 DB이기 때문이다. 이름을 바꾼 뒤 그 이름으로 조회하는 쿼리를 보내면, 변경이 아직 DB에 없는 한 조회 결과에서 그 회원이 빠진다. 그래서 하이버네이트는 쿼리를 보내기 전에 밀린 변경을 먼저 내보낸다.

기준은 SQL이 나가는지가 아니라 조건을 DB에서 평가하는 쿼리인지 여부다. 리포지토리에서는 메서드 이름으로 만들어지는 파생 쿼리와 @Query로 작성한 것이 해당한다. findById는 DB에 SELECT를 보내더라도 해당하지 않는다. 식별자로 엔티티를 찾는 조회라 어긋날 여지가 없다.

readOnly 트랜잭션에서는 FlushMode가 MANUAL이라 커밋 직전과 JPA 쿼리 직전의 flush가 자동으로 일어나지 않는다. 예외로 롤백되면 flush 자체가 없다.

### 커밋 flush는 메서드 본문이 끝난 뒤에 일어난다

```java
@Transactional
public void rename(Long id) {
    Member member = memberRepository.findById(id).orElseThrow();
    member.changeName("홍길동");
}
```

로그를 보면 UPDATE가 본문의 마지막 줄보다 뒤에 있다.

```
Creating new transaction with name [MemberService.rename]
    select ...                                     ← findById
Completing transaction for [MemberService.rename]   ← 본문이 끝난 지점
Initiating transaction commit
    update member set name=? where id=?             ← 본문이 끝난 뒤에 나감
Closing JPA EntityManager after transaction
```

트랜잭션을 커밋하는 주체는 이 메서드가 아니라 이 빈을 감싼 프록시다. 본문이 반환되고 나서 프록시가 커밋하고, 그 과정에서 flush가 일어난다. 프록시가 무엇이고 왜 그것이 트랜잭션 경계를 정하는지는 [[transaction-proxy-boundary]]에서 다룬다.

여기서 실무적인 결과가 하나 나온다. 커밋 단계에서 발생하는 예외는 메서드 안의 try-catch로 잡히지 않는다.

```java
@Transactional
public void rename(Long id) {
    try {
        Member member = memberRepository.findById(id).orElseThrow();
        member.changeName("홍길동");     // 이름에 유니크 제약이 걸려 있다고 하자
    } catch (Exception e) {
        log.error("이름 변경 실패", e);   // 이 로그는 찍히지 않는다
    }
}
```

try 블록 안에서는 객체의 필드만 바뀌고 DB로는 아무것도 나가지 않는다. UPDATE는 본문이 끝난 뒤에 나가고, 제약 위반 예외도 그때 발생한다. 이미 catch 범위를 벗어난 뒤다.

잡으려면 두 가지 방법이 있다.

- 이 메서드를 호출하는 쪽에서 잡는다. 트랜잭션 바깥이므로 거기서는 잡힌다
- entityManager.flush()를 직접 호출해 예외 발생 지점을 앞당긴다. 다만 트랜잭션은 여전히 롤백 대상으로 표시되므로, 잡은 뒤 정상 종료하려는 의도라면 이것만으로는 부족하다

같은 이유로 낙관적 락 충돌도 커밋 시점에 드러난다.

## 상태를 직접 확인하기

DB 반영 여부로 상태를 추측하는 대신 직접 물어볼 수 있다.

```java
@Transactional(readOnly = true)
public void rename(Long id) {
    Member member = memberRepository.findById(id).orElseThrow();
    member.changeName("홍길동");

    System.out.println(entityManager.contains(member));   // true
    System.out.println(member.getName());                 // 홍길동
}
```

contains는 해당 엔티티가 현재 영속성 컨텍스트의 관리 대상인지 알려준다. 세 경우에서 이렇게 갈린다.

- 트랜잭션 없음 — false
- readOnly — true
- 쓰기 — true

같은 자리에서 getName은 세 경우 모두 홍길동을 반환한다. 자바 객체의 필드는 어느 경우에나 바뀌기 때문이다. 객체가 바뀌는 것과 DB가 바뀌는 것은 별개의 사건이다.

이 차이가 실무에서 혼란을 만든다. 변경한 엔티티를 그대로 응답으로 내려주면 화면에는 바뀐 값이 보이고 DB만 그대로다.

## 로그로 구분하기

로그를 켜면 두 원인이 다르게 보인다.

```yaml
logging:
  level:
    org.springframework.orm.jpa.JpaTransactionManager: debug
    org.hibernate.SQL: debug
```

트랜잭션이 없는 경우에는 서비스 클래스 이름이 로그에 없고, 조회 직후 컨텍스트가 닫힌다.

```
Creating new transaction with name [SimpleJpaRepository.findById]: ... ,readOnly
Opened new EntityManager [SessionImpl(156256856)]
    select ...
Closing JPA EntityManager after transaction     ← 이 시점부터 준영속
```

readOnly인 경우에는 서비스 이름이 나오고 뒤에 readOnly가 붙는다. 리포지토리 호출은 새 컨텍스트를 열지 않고 참여한다.

```
Creating new transaction with name [MemberService.rename]: ... ,readOnly
Opened new EntityManager [SessionImpl(204869440)]
    Participating in existing transaction
    select ...
Initiating transaction commit
    (update 없음)
```

판단 순서는 이렇다.

1. Creating new transaction 뒤에 내 서비스 메서드 이름이 있는가. 없으면 트랜잭션 자체가 없는 것이다. 왜 없는지는 [[transaction-proxy-boundary]]를 본다.
2. 있다면 그 줄 끝에 readOnly가 붙어 있는가. 붙어 있으면 flush가 막힌 것이다.

## 준영속인데 반영되는 경우

준영속이어도 save를 호출하면 반영된다. 다만 대가가 있다.

```java
public void rename(Long id) {                                  // 트랜잭션 없음
    Member member = memberRepository.findById(id).orElseThrow();   // select 1
    member.changeName("홍길동");
    memberRepository.save(member);                                 // select 2, 그리고 update
}
```

save는 넘어온 엔티티의 식별자가 null이 아니면 기존 엔티티로 판단하고 merge를 호출한다. merge는 DB에서 현재 상태를 다시 읽어 영속 엔티티를 만든 뒤 값을 복사하는 방식이라 SELECT가 한 번 더 나간다.

그래서 이 메서드는 select를 두 번 실행한다. 목록을 돌면서 같은 패턴을 반복하면 쿼리가 두 배가 된다.

merge에는 주의할 점이 하나 더 있다. 넘어온 객체의 필드를 그대로 덮어쓰므로, 준영속 엔티티의 일부 필드가 비어 있으면 그 값이 null로 저장된다.

정리하면 트랜잭션 경계를 제대로 잡는 것이 먼저이고, save는 준영속 엔티티를 다룰 때만 쓴다.
