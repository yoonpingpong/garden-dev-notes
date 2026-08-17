---
title: "@Transactional과 프록시 경계"
type: concept
tags: [spring, transaction, aop, proxy, self-invocation, jpa]
related: []
last_reviewed: 2026-08-17
publish: true
---

# @Transactional과 프록시 경계

트랜잭션이 걸리는 기준은 애노테이션이 어디 붙었는지가 아니라 그 메서드가 어떤 경로로 호출되었는지다.

## 목차

- [한눈에](#한눈에)
- [개요](#개요)
- [시나리오 — 회원 이름 일괄 변경](#시나리오--회원-이름-일괄-변경)
  - [요구사항](#요구사항)
  - [❌ 같은 클래스 안에서 호출](#-같은-클래스-안에서-호출)
  - [✅ 다른 빈으로 분리](#-다른-빈으로-분리)
  - [무엇이 달라졌나](#무엇이-달라졌나)
- [로그로 확인하기](#로그로-확인하기)
- [같은 제약을 공유하는 애노테이션](#같은-제약을-공유하는-애노테이션)
- [프록시가 아예 만들어지지 않는 경우](#프록시가-아예-만들어지지-않는-경우)

## 한눈에

- 스프링은 @Transactional이 붙은 빈을 프록시로 감싸 컨테이너에 등록한다. 트랜잭션을 여는 코드는 그 프록시에만 있다.
- 프록시는 바깥에서 자기에게 들어온 메서드 하나만 검사한다. 같은 클래스 안에서 부른 메서드는 프록시를 거치지 않으므로 애노테이션이 읽히지 않는다.
- 적용되지 않아도 예외나 경고가 없다. 컴파일도 되고 테스트도 통과한다. 그래서 트랜잭션 로그를 켜고 눈으로 확인해야 한다.
- 안쪽 메서드에 독립된 트랜잭션 설정이 필요하면 그 메서드를 다른 빈으로 분리한다.

## 개요

애노테이션은 명령이 아니라 표시다. 자바 애노테이션 자체는 어떤 코드도 실행하지 않는다. 누군가 그 표시를 읽고 행동해야 의미가 생긴다.

스프링에서 그 역할을 하는 것이 프록시다. 애플리케이션이 기동할 때 스프링은 @Transactional이 붙은 클래스를 상속한 가짜 클래스를 만들고, 그 가짜 객체를 컨테이너에 등록한다. 진짜 객체는 프록시 안에 감춰진다.

```
컨테이너
  "memberService" → MemberService$$SpringCGLIB$$0 인스턴스
                        └─ target ─→ MemberService 인스턴스
```

프록시는 호출을 가로채 트랜잭션을 열고, 진짜 객체의 메서드를 실행하고, 커밋하거나 롤백한다. 그래서 프록시를 지나지 않은 호출에는 트랜잭션이 생기지 않는다.

같은 클래스 안에서 자기 메서드를 호출하는 것을 self-invocation이라고 부른다. 이 호출은 프록시 바깥이 아니라 안쪽에서 일어나므로 프록시가 관여할 수 없다. 스프링에서 애노테이션이 무시되는 사례의 대부분이 여기에 해당한다.

애노테이션을 붙였다고 해서 원래 메서드가 바뀌지는 않는다. 컴파일된 rename 메서드 안에는 조회하고 값을 바꾸는 코드만 있고 트랜잭션을 여는 코드는 한 줄도 없다. 메서드는 그대로 있고 꼬리표만 붙은 것이다. 그 꼬리표를 읽어 주는 프록시를 통과하는지가 전부를 결정한다.

## 시나리오 — 회원 이름 일괄 변경

### 요구사항

여러 회원의 이름을 한 번에 바꾼다. 변경 대상은 아래와 같이 주어진다.

```java
Map<Long, String> newNames = Map.of(
    1L, "홍길동",
    2L, "김철수"
);
```

이름 변경은 JPA의 변경 감지에 맡긴다. 엔티티를 조회해 값을 바꾸면 트랜잭션이 커밋될 때 UPDATE가 나간다. save를 따로 호출하지 않는다.

### ❌ 같은 클래스 안에서 호출

```java
@Service
public class MemberService {

    private final MemberRepository memberRepository;

    public MemberService(MemberRepository memberRepository) {
        this.memberRepository = memberRepository;
    }

    public void renameAll(Map<Long, String> newNames) {
        newNames.forEach((id, name) -> rename(id, name));   // 같은 클래스 안의 메서드
    }

    @Transactional
    public void rename(Long id, String name) {
        Member member = memberRepository.findById(id).orElseThrow();
        member.changeName(name);
    }
}
```

호출하면 아무 일도 일어나지 않는다. 예외는 나지 않고, UPDATE도 나가지 않고, DB의 이름은 그대로다. 테스트를 써도 통과한다.

renameAll이 실행되는 시점에는 이미 프록시를 통과해 진짜 객체 안에 들어와 있다. 거기서 부르는 rename은 진짜 객체가 자기 메서드를 부르는 평범한 자바 호출이다. 프록시는 이 호출을 볼 방법이 없다.

트랜잭션이 없으므로 findById가 반환하는 순간 영속성 컨텍스트가 닫히고 엔티티는 준영속이 된다. 그 뒤의 changeName은 아무도 추적하지 않는 객체의 필드를 바꾼 것이다.

### ✅ 다른 빈으로 분리

```java
@Service
public class MemberService {

    private final MemberRenamer memberRenamer;

    public MemberService(MemberRenamer memberRenamer) {
        this.memberRenamer = memberRenamer;
    }

    public void renameAll(Map<Long, String> newNames) {
        newNames.forEach(memberRenamer::rename);            // 다른 빈의 메서드
    }
}

@Service
public class MemberRenamer {

    private final MemberRepository memberRepository;

    public MemberRenamer(MemberRepository memberRepository) {
        this.memberRepository = memberRepository;
    }

    @Transactional
    public void rename(Long id, String name) {
        Member member = memberRepository.findById(id).orElseThrow();
        member.changeName(name);
    }
}
```

memberRenamer는 주입받은 프록시다. 호출이 프록시를 지나므로 애노테이션이 읽히고 트랜잭션이 열린다. 건마다 UPDATE가 나간다.

### 무엇이 달라졌나

- rename 메서드의 본문은 한 글자도 바뀌지 않았다. 애노테이션도 그대로다.
- 달라진 것은 호출 대상이다. 앞에서는 MemberService의 진짜 객체가 자기 자신의 rename을 불렀고, 뒤에서는 같은 객체가 MemberRenamer의 프록시를 부른다.
- 그 차이 하나로 트랜잭션이 생기고, 엔티티가 영속 상태로 유지되고, 변경 감지가 동작한다.

## 로그로 확인하기

트랜잭션 경계를 로그에 남기면 눈으로 확인할 수 있다.

```yaml
logging:
  level:
    org.springframework.transaction.interceptor.TransactionInterceptor: trace
    org.springframework.orm.jpa.JpaTransactionManager: debug
    org.hibernate.SQL: debug
```

분리하기 전에는 서비스 클래스 이름이 어디에도 나오지 않는다.

```
Getting transaction for [SimpleJpaRepository.findById]
Completing transaction for [SimpleJpaRepository.findById]
```

리포지토리 메서드에는 Spring Data가 트랜잭션을 걸어 두었으므로 그것만 찍힌다. 조회는 되지만 조회가 끝나는 순간 트랜잭션도 끝난다.

분리한 뒤에는 이렇게 바뀐다.

```
Creating new transaction with name [MemberRenamer.rename]
Opened new EntityManager [SessionImpl(156256856)]
    Found thread-bound EntityManager [SessionImpl(156256856)]
    Participating in existing transaction
    select ...
Completing transaction for [MemberRenamer.rename]
Initiating transaction commit
    update member set name=? where id=?
```

두 가지를 본다.

- Creating new transaction 뒤에 내가 의도한 메서드 이름이 있는가. 없으면 프록시를 거치지 않았거나 애노테이션이 없는 것이다.
- 리포지토리 호출이 Participating in existing transaction으로 바뀌었는가. 바뀌었다면 하나의 트랜잭션 안에 들어온 것이다.

Participating 위의 Found thread-bound EntityManager가 원리를 보여준다. 스프링은 트랜잭션을 시작할 때 EntityManager를 현재 스레드에 묶어 두고, 같은 스레드의 이후 호출이 그것을 찾아 재사용한다. 스레드가 바뀌면 이 연결이 끊어진다. parallelStream이나 @Async에서 트랜잭션이 전파되지 않는 이유가 여기에 있다.

## 같은 제약을 공유하는 애노테이션

프록시로 동작하는 기능은 전부 같은 제약을 갖는다. 내부 호출에서는 무력화된다.

- @Async — 별도 스레드로 실행되지 않고 호출한 스레드에서 그대로 돈다
- @Cacheable — 캐시를 확인하지 않고 매번 원본 메서드가 실행된다
- @PreAuthorize — 권한 검사가 일어나지 않는다
- @Retryable — 재시도가 동작하지 않는다

"애노테이션을 붙였는데 안 먹는다"는 증상이 나오면 호출 경로부터 확인한다.

전파 옵션도 마찬가지다. 아래 코드에서 REQUIRES_NEW는 읽히지도 않는다.

```java
@Transactional
public void settleAll(List<Long> ids) {
    ids.forEach(this::settleOne);
}

@Transactional(propagation = Propagation.REQUIRES_NEW)   // 무시된다
public void settleOne(Long id) { ... }
```

바깥에 트랜잭션이 있으니 동작은 한다. 그래서 문제가 없어 보인다. 작성자는 건별로 독립된 트랜잭션을 의도했지만 실제로는 전부 하나로 묶인다. 한 건이 실패하면 전체가 롤백된다.

## 프록시가 아예 만들어지지 않는 경우

호출 경로가 맞는데도 동작하지 않는다면 프록시를 만들 수 없는 형태일 수 있다. 스프링 부트는 대상 클래스를 상속하는 방식으로 프록시를 만들므로, 상속과 재정의가 불가능한 것에는 걸리지 않는다.

- private 메서드 — 상속되지 않아 재정의 대상이 아니다
- final 메서드 — 재정의가 금지되어 있다
- final 클래스 — 상속 자체가 불가능해 프록시 생성이 실패한다
- static 메서드 — 호출 대상이 컴파일 시점에 확정되어 가로챌 수 없다

트랜잭션을 걸 메서드는 public으로 두는 것이 안전하다.
