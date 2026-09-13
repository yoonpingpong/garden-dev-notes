---
title: "@EnableJpaAuditing을 별도 설정 클래스에 두는 이유"
type: pattern
tags: [spring, spring-boot, configuration, jpa-auditing, slice-test, test, enable]
related:
  - "[[bean-registration-paths]]"
  - "[[persistence-state-and-flush]]"
last_reviewed: 2026-09-13
publish: true
---

# @EnableJpaAuditing을 별도 설정 클래스에 두는 이유

기능을 켜는 애노테이션을 메인 애플리케이션 클래스에 붙이면 모든 테스트가 그 설정을 함께 읽는다. JPA가 필요 없는 슬라이스 테스트까지 JPA 설정을 읽게 되고, 거기서 실패한다. 이 노트는 Spring Boot 4.1.1, Java 21을 기준으로 쓴다.

## 목차

- [개요](#개요)
- [시나리오 — 웹 슬라이스 테스트가 깨진다](#시나리오--웹-슬라이스-테스트가-깨진다)
  - [요구사항](#요구사항)
  - [❌ 메인 클래스에 붙인 경우](#-메인-클래스에-붙인-경우)
  - [✅ 별도 설정 클래스로 분리한 경우](#-별도-설정-클래스로-분리한-경우)
  - [무엇이 달라졌나](#무엇이-달라졌나)
- [감사가 필요한 테스트에서 가져오기](#감사가-필요한-테스트에서-가져오기)
- [다른 기능 활성화 애노테이션도 마찬가지다](#다른-기능-활성화-애노테이션도-마찬가지다)
- [관련 노트](#관련-노트)

## 개요

메인 애플리케이션 클래스는 모든 테스트가 설정의 출발점으로 삼는 자리다. @SpringBootTest는 물론이고 @WebMvcTest나 @DataJpaTest 같은 슬라이스 테스트도 @SpringBootConfiguration이 붙은 클래스를 찾아 올라가 거기서부터 설정을 읽는다.

그래서 메인 클래스에 붙인 애노테이션은 어떤 테스트에서든 함께 활성화된다. 슬라이스 테스트는 필요한 계층만 올리려고 쓰는 것인데, 메인 클래스에 붙은 애노테이션은 그 선별을 통과해 버린다. 활성화된 기능이 그 슬라이스에 없는 빈을 요구하면 컨텍스트 로드 단계에서 실패한다.

## 시나리오 — 웹 슬라이스 테스트가 깨진다

### 요구사항

- 엔티티에 생성 시각과 수정 시각을 자동으로 채운다. JPA 감사 기능이 필요하다.
- 컨트롤러만 검증하는 웹 슬라이스 테스트가 있다. 이 테스트는 데이터베이스를 쓰지 않는다.
- 두 가지가 서로를 방해하지 않아야 한다.

### ❌ 메인 클래스에 붙인 경우

감사 기능을 켜는 가장 짧은 방법은 메인 클래스에 애노테이션을 한 줄 더하는 것이다.

```java
@SpringBootApplication
@EnableJpaAuditing
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

애플리케이션 실행에는 문제가 없다. 깨지는 것은 웹 슬라이스 테스트다.

```java
@WebMvcTest(MemberController.class)
class MemberControllerTest {
    // JPA를 쓰지 않지만 @EnableJpaAuditing이 함께 활성화된다
}
```

@EnableJpaAuditing이 활성화되면 스프링은 jpaAuditingHandler 빈을 등록한다. 이 빈은 jpaMappingContext를 생성자 인자로 요구하고, jpaMappingContext는 JPA 메타모델이 비어 있으면 만들어지지 않는다. 웹 슬라이스에는 엔티티가 올라오지 않으므로 메타모델이 비어 있고, 그 지점에서 컨텍스트 로드가 무너진다.

실제로 확인한 예외 연쇄다. 클래스명은 일반적인 이름으로 바꾸었다.

```text
java.lang.IllegalStateException: Failed to load ApplicationContext for
    [WebMergedContextConfiguration ... testClass = MemberControllerTest ...]
  BeanCreationException: Error creating bean with name 'jpaAuditingHandler':
      Cannot resolve reference to bean 'jpaMappingContext' while setting constructor argument
    BeanCreationException: Error creating bean with name 'jpaMappingContext':
        JPA metamodel must not be empty
      IllegalArgumentException: JPA metamodel must not be empty
```

@EnableJpaAuditing을 메인 클래스로 옮기고 웹 슬라이스 테스트를 실행하니 테스트 네 개가 모두 실패했고, 원래 위치로 되돌리자 다시 통과했다. 네 개가 각각 다른 이유로 실패한 것이 아니다. 애플리케이션 컨텍스트 로드 자체가 실패했으므로 그 컨텍스트를 쓰는 테스트가 전부 같은 원인으로 무너진 것이다.

### ✅ 별도 설정 클래스로 분리한 경우

애노테이션을 전용 설정 클래스로 옮긴다.

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

```java
@SpringBootApplication
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

웹 슬라이스 테스트는 그대로 통과한다. 달라진 것은 애노테이션이 붙은 위치다. 메인 클래스는 슬라이스 테스트가 설정을 읽기 시작하는 출발점이라 거기 붙은 애노테이션은 어떤 슬라이스에서든 함께 활성화되지만, 별도 설정 클래스는 명시적으로 가져올 때만 활성화된다. @WebMvcTest가 컴포넌트 스캔 대상을 웹 계층 관련 클래스로 제한하는 것이 그 활성화를 막는 구체적인 장치다.

이 구분을 흐리면 잘못된 결론에 이른다. 일반 @Configuration이 슬라이스에서 스캔되지 않는다는 사실만 근거로 삼으면, 감사 설정이 어떤 이유로든 다시 활성화되는 상황에서 같은 실패가 나는 것을 설명하지 못한다. 실패를 가른 것은 스캔 여부가 아니라 애노테이션의 위치다.

애플리케이션을 정상 기동할 때는 컴포넌트 스캔이 이 클래스를 찾으므로 감사 기능이 켜진다.

### 무엇이 달라졌나

| | 메인 클래스에 붙임 | 별도 설정 클래스 |
|---|---|---|
| 애플리케이션 기동 | 감사 기능 켜짐 | 감사 기능 켜짐 |
| 웹 슬라이스 테스트 | 감사 빈이 활성화되어 컨텍스트 로드 실패 | 활성화되지 않아 통과 |
| JPA 슬라이스 테스트 | 항상 활성화 | 필요할 때 명시적으로 가져옴 |
| 메인 클래스 | 기능이 늘수록 애노테이션이 쌓임 | @SpringBootApplication 한 줄 유지 |

## 감사가 필요한 테스트에서 가져오기

분리하면 기본적으로 읽히지 않으므로, 감사 동작을 검증하는 테스트에서는 명시적으로 가져온다.

```java
@DataJpaTest
@Import(JpaAuditingConfig.class)
class AuditableEntityTest {
    // 생성 시각과 수정 시각이 채워지는지 검증한다
}
```

가져오지 않은 @DataJpaTest에서는 감사 필드가 채워지지 않는다. 그 테스트가 생성 시각을 단언하고 있다면 값이 null이라 실패한다. 분리의 대가는 이 한 줄을 기억해야 한다는 것이다.

## 다른 기능 활성화 애노테이션도 마찬가지다

같은 이유가 다른 기능에도 적용된다. @EnableScheduling을 메인 클래스에 두면 모든 테스트에서 스케줄러가 뜨고, 테스트 실행 중에 예정된 작업이 실제로 돌아간다. @EnableAsync도 비슷하게 테스트의 실행 흐름을 바꾼다.

기능마다 전용 설정 클래스를 두는 것이 일반적인 구성이다. 부수적인 이점으로 메인 클래스에 애노테이션이 쌓이지 않는다. 메인 클래스는 애플리케이션의 진입점이지 설정을 모아 두는 자리가 아니다.

```
config/
├── JpaAuditingConfig.java     @EnableJpaAuditing
├── SchedulingConfig.java      @EnableScheduling
└── AsyncConfig.java           @EnableAsync
```

## 관련 노트

- [[bean-registration-paths]] — 설정 클래스가 무엇을 선언하며 일반 컴포넌트와 어떻게 다른가
- [[persistence-state-and-flush]] — 감사 필드가 실제로 채워지는 시점을 결정하는 영속성 컨텍스트 동작
