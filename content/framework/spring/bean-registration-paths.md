---
title: 빈을 등록하는 두 경로
type: concept
tags: [spring, spring-boot, configuration, bean, component, proxy, di]
related:
  - "[[enable-annotation-placement]]"
  - "[[transaction-proxy-boundary]]"
last_reviewed: 2026-09-13
publish: true
---

# 빈을 등록하는 두 경로

스프링 컨테이너에 빈을 올리는 방법은 클래스에 애노테이션을 붙이는 것과 설정 클래스 안에서 메서드로 만드는 것 두 가지다. 이 노트는 둘을 가르는 기준과, 설정 클래스가 일반 컴포넌트와 무엇이 다른지를 정리한다. 이 노트는 Spring Boot 4.1.1, Java 21을 기준으로 쓴다.

## 목차

- [개요](#개요)
- [Configuration이 선언하는 것](#configuration이-선언하는-것)
- [경로 1 — 클래스에 직접 붙이기](#경로-1--클래스에-직접-붙이기)
- [경로 2 — 설정 클래스 안에서 메서드로 만들기](#경로-2--설정-클래스-안에서-메서드로-만들기)
- [스위치와 부품을 한 클래스에 모으기](#스위치와-부품을-한-클래스에-모으기)
- [Component와 무엇이 다른가](#component와-무엇이-다른가)
- [관련 노트](#관련-노트)

## 개요

두 경로를 가르는 질문은 하나다. 그 클래스에 애노테이션을 붙일 수 있는가. 내가 작성한 클래스라면 붙일 수 있고, 라이브러리가 제공하는 클래스라면 붙일 수 없다. 붙일 수 없는 것을 빈으로 만들려면 만드는 코드를 내가 써야 하고, 그 코드가 들어가는 자리가 설정 클래스의 @Bean 메서드다.

## Configuration이 선언하는 것

@Configuration은 이 클래스가 빈을 만드는 방법을 담고 있다는 선언이다. 과거에 XML 파일로 적던 설정을 자바 코드로 옮긴 자리이며, 그래서 XML의 beans 태그가 하던 일을 그대로 물려받는다.

인터페이스에 구현체를 지정해 빈으로 만드는 것은 그중 한 가지 용도일 뿐이다. 그것만으로 @Configuration을 이해하면 범위가 좁아진다. 설정 클래스에는 구현체 선택뿐 아니라 외부 라이브러리 객체의 조립, 프로퍼티 값에 따른 분기, 기능 활성화 애노테이션까지 들어간다.

## 경로 1 — 클래스에 직접 붙이기

@Component와 그 특수화인 @Service, @Repository, @Controller를 클래스에 붙이면 컴포넌트 스캔이 그 클래스를 찾아 빈으로 등록한다.

```java
@Service
public class MemberService {
    // 스프링이 이 클래스를 찾아 인스턴스를 만들고 컨테이너에 넣는다
}
```

조건은 하나다. 그 클래스의 소스가 내 프로젝트에 있어야 한다. 애노테이션은 소스에 적는 것이므로, 남이 만든 jar 안의 클래스에는 붙일 수 없다.

@Service와 @Repository는 @Component에 의미를 덧붙인 것이다. @Repository는 추가 동작이 있어서, 구현체가 던지는 데이터 접근 예외를 스프링의 DataAccessException 계층으로 변환한다. @Service는 현재 추가 동작 없이 역할 표시만 한다.

## 경로 2 — 설정 클래스 안에서 메서드로 만들기

@Configuration 클래스 안에 @Bean을 붙인 메서드를 두면, 그 메서드의 반환값이 빈으로 등록된다. 빈 이름은 기본적으로 메서드 이름이다.

이 경로를 쓰는 경우는 세 가지다.

첫째, 내 소유가 아닌 클래스일 때다. 라이브러리가 제공하는 타입에는 애노테이션을 붙일 수 없으므로 만드는 코드를 직접 쓴다.

```java
@Configuration
public class JacksonConfig {

    @Bean
    public ObjectMapper objectMapper() {
        return JsonMapper.builder()
                .addModule(new JavaTimeModule())
                .build();
    }
}
```

둘째, 생성 과정에 로직이 필요할 때다. 설정값에 따라 구현을 바꾸거나 여러 값을 조합해 조립하는 경우가 여기 해당한다. 애노테이션 한 줄로는 표현할 수 없는 절차가 있으면 메서드 본문이 필요하다.

```java
@Configuration
public class RetryConfig {

    @Bean
    public Duration retryInterval(@Value("${app.retry-seconds:3}") int seconds) {
        return Duration.ofSeconds(seconds);
    }
}
```

셋째, 구현이 짧아 클래스를 따로 만들 이유가 없을 때다. 람다 한 줄로 끝나는 인터페이스 구현을 위해 파일을 하나 더 만드는 것은 과하다.

## 스위치와 부품을 한 클래스에 모으기

기능을 켜는 애노테이션과 그 기능이 요구하는 빈을 한 설정 클래스에 함께 두는 구성이 흔하다. JPA 감사 기능이 그 예다.

```java
@Configuration
@EnableJpaAuditing
public class JpaAuditingConfig {

    @Bean
    public AuditorAware<String> auditorAware() {
        return () -> Optional.of("system");
    }
}
```

@EnableJpaAuditing이 기능을 켜는 스위치이고, AuditorAware는 그 기능이 작성자 이름을 물어볼 때 답하는 부품이다. 둘을 한 클래스에 두면 이 기능에 관한 설정이 한 파일에 모인다. 위 구현은 항상 같은 문자열을 돌려주므로 실제 서비스에서는 인증 주체를 읽는 코드로 바꾼다.

이 클래스를 메인 애플리케이션 클래스와 분리해서 두는 데에는 별도의 이유가 있다. [[enable-annotation-placement]]에서 다룬다.

## Component와 무엇이 다른가

@Component에도 @Bean 메서드를 쓸 수 있다. 그러나 @Configuration에는 추가 동작이 있다. @Bean 메서드끼리 서로 호출할 때, @Configuration은 새 객체를 만들지 않고 이미 등록된 빈을 돌려준다.

```java
@Configuration
public class CounterConfig {

    @Bean
    public AtomicInteger counter() {
        return new AtomicInteger(0);
    }

    @Bean
    public IntSupplier counterReader() {
        return counter()::get;   // 같은 클래스의 @Bean 메서드를 직접 호출한다
    }
}
```

이 설정에서 counter 빈의 값을 올린 뒤 counterReader로 읽으면 결과가 갈린다.

```java
counter.incrementAndGet();   // 등록된 counter 빈의 값을 1로 올린다
counterReader.getAsInt();    // @Configuration이면 1, @Component면 0
```

@Component에서는 counter() 호출이 평범한 자바 메서드 호출이라 new AtomicInteger(0)이 한 번 더 실행된다. counterReader는 컨테이너에 등록된 것과 다른 객체를 들여다보게 되고, 값은 0에 머문다.

@Configuration이 다르게 동작하는 이유는 프록시다. 스프링은 @Configuration 클래스를 상속한 프록시를 만들어 컨테이너에 등록하고, 그 프록시가 @Bean 메서드 호출을 가로채 이미 만들어 둔 빈이 있으면 그것을 돌려준다. 그래서 빈 사이에 의존이 있어도 싱글턴이 유지된다. 프록시가 호출을 가로채는 구조 자체는 [[transaction-proxy-boundary]]와 같다.

이 동작에는 두 가지 단서가 붙는다. 프록시를 만들기 위해 상속이 필요하므로 @Configuration 클래스와 @Bean 메서드는 final이면 안 된다. 그리고 proxyBeanMethods 속성을 false로 주면 프록시를 만들지 않아 @Component와 같은 동작이 되는데, 기동 속도를 위해 이 값을 끄는 설정이 스프링 내부에 실제로 존재한다. @Bean 메서드끼리 호출하지 않는 설정 클래스에서만 안전하다.

## 관련 노트

- [[enable-annotation-placement]] — 기능 활성화 애노테이션을 어느 설정 클래스에 둘 것인가
- [[transaction-proxy-boundary]] — 스프링이 프록시로 애노테이션을 처리하는 구조
