---
title: 의존성 configuration 구분
type: concept
tags: [gradle, kotlin-dsl, spring-boot, implementation, compileOnly, runtimeOnly, annotationProcessor, lombok]
related:
  - "[[overview]]"
  - "[[plugins-vs-dependencies]]"
  - "[[bom-version-management]]"
last_reviewed: 2026-09-13
publish: false
---

# 의존성 configuration 구분

dependencies 블록에서 좌표 앞에 붙는 implementation, compileOnly, runtimeOnly 같은 이름을 Gradle은 configuration이라고 부른다. 이 이름이 그 라이브러리가 컴파일과 실행 중 어느 시점에 클래스패스에 올라가는지를 정한다. 이 노트는 Spring Boot 4.1.1, Gradle Kotlin DSL을 기준으로 쓴다. dependencies 블록 자체의 역할은 [[plugins-vs-dependencies]] 참조.

## 목차

- [개요](#개요)
- [configuration별 구분](#configuration별-구분)
  - [implementation](#implementation)
  - [compileOnly](#compileonly)
  - [runtimeOnly](#runtimeonly)
  - [testImplementation](#testimplementation)
  - [annotationProcessor](#annotationprocessor)
- [롬복이 두 줄인 이유](#롬복이-두-줄인-이유)
- [관련 노트](#관련-노트)

## 개요

start.spring.io가 만든 프로젝트의 dependencies 블록은 다음과 같은 모양이다.

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    compileOnly("org.projectlombok:lombok")
    annotationProcessor("org.projectlombok:lombok")
    runtimeOnly("com.h2database:h2")
    testImplementation("org.springframework.boot:spring-boot-starter-test")
}
```

다섯 줄이 네 가지 시점 조합을 쓴다. 구분 기준은 두 가지 질문이다. 내 코드를 컴파일할 때 이 라이브러리가 필요한가, 그리고 애플리케이션을 실행할 때 이 라이브러리가 필요한가.

| configuration | 컴파일 시 | 실행 시 | 최종 jar 포함 |
|---|---|---|---|
| implementation | 필요 | 필요 | 포함 |
| compileOnly | 필요 | 불필요 | 미포함 |
| runtimeOnly | 불필요 | 필요 | 포함 |
| testImplementation | 테스트 컴파일 시 | 테스트 실행 시 | 미포함 |
| annotationProcessor | 컴파일 도구로 실행 | 불필요 | 미포함 |

## configuration별 구분

### implementation

컴파일할 때도 실행할 때도 필요한 라이브러리다. 대부분의 의존성이 여기 속한다. spring-boot-starter-web을 implementation으로 넣으면 컨트롤러 코드가 스프링 MVC 애노테이션을 임포트할 수 있고, 실행 시 내장 톰캣이 함께 뜬다.

implementation이라는 이름에는 구현 세부사항을 감춘다는 뜻도 있다. 이 뜻은 모듈이 여러 개일 때만 드러난다. 모듈 A가 어떤 라이브러리를 implementation으로 선언하면, A를 의존하는 모듈 B의 컴파일 클래스패스에는 그 라이브러리가 전파되지 않는다. B가 그 라이브러리를 직접 쓰려면 B의 dependencies에 따로 적어야 한다. 반대로 A가 api로 선언하면 B의 컴파일 클래스패스까지 전파된다. api configuration은 java 플러그인에는 없고 java-library 플러그인을 적용해야 생긴다.

implementation을 기본으로 쓰는 이유는 두 가지다. 캡슐화 측면에서 A가 내부적으로 쓰는 라이브러리가 A의 공개 API에 섞이지 않는다. 빌드 속도 측면에서 A의 내부 의존성이 바뀌어도 B를 다시 컴파일할 필요가 없다. 과거의 compile configuration은 모든 의존성을 전파해 이 두 문제를 함께 안고 있었고, Gradle은 이것을 implementation과 api로 갈라 해결했다.

단일 모듈 프로젝트에서는 전파받을 모듈이 없으므로 implementation과 api의 차이가 드러나지 않는다. 이 노트의 다른 절은 단일 모듈을 전제로 쓴다.

### compileOnly

컴파일할 때만 필요하고 실행할 때는 필요 없는 라이브러리다. 롬복이 대표적이다. 롬복은 컴파일 시점에 게터와 세터 코드를 생성하고 끝난다. 생성된 코드는 일반 자바 코드이므로 실행 시 롬복 라이브러리가 없어도 동작한다. 따라서 jar에 넣을 이유가 없다.

### runtimeOnly

컴파일할 때는 필요 없고 실행할 때만 필요한 라이브러리다. H2 데이터베이스 드라이버가 그렇다. 내 코드는 java.sql 패키지의 JDBC 표준 인터페이스만 쓰고, org.h2 패키지의 드라이버 클래스를 직접 임포트하지 않는다. 컴파일러는 인터페이스만 보면 되고, 실제 구현 클래스는 실행 시 JDBC가 클래스패스에서 찾는다.

이 구분이 주는 이점이 있다. 드라이버를 runtimeOnly로 두면 코드 어디에서도 H2 전용 클래스를 참조할 수 없다. 참조하면 컴파일 에러가 나기 때문이다. 나중에 PostgreSQL로 바꿀 때 runtimeOnly 한 줄만 교체하면 되는 상태가 강제된다.

### testImplementation

테스트 코드에서만 쓰는 라이브러리다. src/test 아래 코드를 컴파일하고 실행할 때만 클래스패스에 오른다. spring-boot-starter-test를 여기 두면 JUnit과 Mockito가 운영 산출물에 섞이지 않는다.

### annotationProcessor

컴파일 과정에서 코드를 생성하는 도구를 지정한다. 자바 컴파일러는 소스를 읽는 동안 애노테이션을 만나면 등록된 프로세서를 호출하고, 프로세서는 그 자리에 코드를 생성할 수 있다. annotationProcessor에 적은 라이브러리는 컴파일러의 도구로 실행되지 컴파일 대상 코드에 포함되는 것이 아니다.

## 롬복이 두 줄인 이유

위 예시에서 롬복만 compileOnly와 annotationProcessor 두 줄에 등장한다. 두 줄이 서로 다른 일을 맡기 때문이다.

```kotlin
compileOnly("org.projectlombok:lombok")          // 소스가 @Getter 애노테이션 타입을 참조할 수 있게
annotationProcessor("org.projectlombok:lombok")  // 컴파일러가 게터 코드를 실제로 생성하게
```

소스 코드에 @Getter를 적으면 컴파일러는 그 애노테이션 타입이 어디 있는지 알아야 한다. 이것을 compileOnly가 해결한다. 그러나 애노테이션 타입을 아는 것과 게터를 생성하는 것은 별개다. 생성은 컴파일러에 프로세서로 등록되어야 일어나고, 이것을 annotationProcessor가 해결한다.

한 줄만 있으면 각각 다르게 실패한다. compileOnly만 있으면 컴파일은 통과하지만 게터가 생성되지 않아 호출하는 쪽에서 "cannot find symbol" 에러가 난다. annotationProcessor만 있으면 프로세서는 등록되었지만 소스가 @Getter 타입을 찾지 못해 "package lombok does not exist" 에러가 난다.

## 관련 노트

- [[overview]] — 애플리케이션 개발자가 Gradle에서 알아야 할 첫 단계로 이 구분을 꼽는 이유
- [[plugins-vs-dependencies]] — dependencies 블록이 plugins 블록과 어떻게 다른가
- [[bom-version-management]] — 이 블록의 좌표에 버전이 비어 있어도 되는 이유
