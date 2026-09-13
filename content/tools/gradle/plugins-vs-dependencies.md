---
title: plugins와 dependencies의 차이
type: concept
tags: [gradle, kotlin-dsl, spring-boot, plugins, dependencies, build]
related:
  - "[[overview]]"
  - "[[bom-version-management]]"
  - "[[dependency-configurations]]"
last_reviewed: 2026-09-13
publish: false
---

# plugins와 dependencies의 차이

build.gradle.kts의 두 블록은 겉모습이 비슷하지만 가리키는 대상이 다르다. plugins는 빌드를 수행하는 도구를 확장하고, dependencies는 만들어질 애플리케이션이 쓸 라이브러리를 지정한다. 이 노트는 Spring Boot 4.1.1, Java 21, Gradle Kotlin DSL을 기준으로 쓴다. Gradle 전반의 배경은 [[overview]] 참조.

## 목차

- [개요](#개요)
- [plugins는 기계, dependencies는 부품](#plugins는-기계-dependencies는-부품)
- [plugins 블록](#plugins-블록)
- [dependencies 블록](#dependencies-블록)
- [헷갈리는 지점](#헷갈리는-지점)
- [관련 노트](#관련-노트)

## 개요

start.spring.io가 만들어 주는 build.gradle.kts는 다음 두 블록으로 시작한다.

```kotlin
plugins {
    java
    id("org.springframework.boot") version "4.1.1"
    id("io.spring.dependency-management") version "1.1.7"
}

dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    compileOnly("org.projectlombok:lombok")
    annotationProcessor("org.projectlombok:lombok")
    runtimeOnly("com.h2database:h2")
    testImplementation("org.springframework.boot:spring-boot-starter-test")
}
```

두 블록 모두 외부에서 무언가를 가져오는 것처럼 보인다. 그러나 plugins가 가져오는 것은 Gradle 자신을 위한 것이고, dependencies가 가져오는 것은 내 코드를 위한 것이다. 이 구분을 놓치면 "스프링 부트가 왜 두 번 나오는가" 같은 질문에서 막힌다.

## plugins는 기계, dependencies는 부품

공장에 빗대면 plugins는 기계이고 dependencies는 제품에 들어가는 부품이다. 기계는 제품을 만들 때만 쓰이고 제품 안에 들어가지 않는다. 부품은 제품 안에 들어가 출하된다.

이 비유는 "산출물에 포함되는가"라는 축에서만 성립한다. plugins가 dependencies에 영향을 주는 경우가 있는데, 그 지점은 [헷갈리는 지점](#헷갈리는-지점)에서 따로 다룬다.

## plugins 블록

plugins에 적은 것은 Gradle에 태스크와 관례를 추가한다. 위 예시의 세 플러그인은 각각 다음 일을 한다.

| 플러그인 | 추가하는 것 |
|---|---|
| java | compileJava, test, jar 태스크. src/main/java 같은 디렉터리 관례 |
| org.springframework.boot | bootRun, bootJar 태스크. 실행 가능한 jar를 만드는 방법 |
| io.spring.dependency-management | 버전이 비어 있는 의존성에 BOM 기준으로 버전을 채우는 기능 |

java 플러그인이 없으면 Gradle은 자바 소스를 컴파일하는 방법을 모른다. compileJava라는 태스크 자체가 존재하지 않기 때문이다. org.springframework.boot 플러그인이 없으면 ./gradlew bootRun은 "Task 'bootRun' not found"로 실패한다.

세 플러그인 모두 빌드하는 동안만 동작하고, 최종 jar 안에는 들어가지 않는다. io.spring.dependency-management가 하는 일은 [[bom-version-management]]에서 다룬다.

## dependencies 블록

dependencies에 적은 것은 내 코드가 참조하거나 실행 시 필요로 하는 라이브러리다. spring-boot-starter-web을 넣으면 스프링 MVC와 내장 톰캣이 클래스패스에 들어오고, bootJar가 만든 jar 안에도 함께 담긴다.

같은 블록 안에서도 implementation, compileOnly, runtimeOnly처럼 앞에 붙는 이름이 다르다. 이 이름은 그 라이브러리가 컴파일과 실행 중 어느 시점에 필요한지를 가른다. 자세한 구분은 [[dependency-configurations]] 참조.

## 헷갈리는 지점

스프링 부트가 양쪽에 다 등장한다. plugins의 org.springframework.boot는 jar를 만들고 애플리케이션을 실행하는 도구다. dependencies의 spring-boot-starter-web은 애플리케이션이 실행될 때 쓰이는 코드다. 이름이 비슷할 뿐 역할이 다르다.

version의 의미도 다르다. plugins의 version "4.1.1"은 플러그인 자체의 버전이다. dependencies 쪽 좌표에는 버전이 비어 있는데, 이것은 io.spring.dependency-management 플러그인이 BOM을 읽어 채워 주기 때문이다. 즉 plugins 블록에 있는 플러그인이 dependencies 블록의 내용을 바꾼다. 공장 비유가 여기서 끊어진다. 기계가 부품 목록의 빈칸을 채우는 셈이다.

## 관련 노트

- [[overview]] — Gradle 스크립트가 설정 파일이 아니라 프로그램이라는 전제
- [[bom-version-management]] — dependencies의 빈 버전을 누가 어떻게 채우는가
- [[dependency-configurations]] — implementation, compileOnly, runtimeOnly의 구분
