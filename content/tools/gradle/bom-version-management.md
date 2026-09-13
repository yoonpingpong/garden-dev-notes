---
title: BOM으로 의존성 버전 관리하기
type: concept
tags: [gradle, kotlin-dsl, spring-boot, bom, dependency-management, version]
related:
  - "[[overview]]"
  - "[[plugins-vs-dependencies]]"
  - "[[dependency-configurations]]"
last_reviewed: 2026-09-13
publish: false
---

# BOM으로 의존성 버전 관리하기

스프링 부트 프로젝트의 dependencies 블록에는 버전이 거의 적혀 있지 않다. 그래도 빌드가 되는 이유는 io.spring.dependency-management 플러그인이 BOM을 읽어 빈 버전을 채워 주기 때문이다. 이 노트는 Spring Boot 4.1.1, Gradle Kotlin DSL을 기준으로 쓴다. plugins와 dependencies의 역할 구분은 [[plugins-vs-dependencies]] 참조.

## 목차

- [개요](#개요)
- [BOM이란](#bom이란)
- [시나리오 — 플러그인을 빼고 빌드하면](#시나리오--플러그인을-빼고-빌드하면)
  - [❌ 플러그인 없이](#-플러그인-없이)
  - [✅ 플러그인과 함께](#-플러그인과-함께)
- [이점](#이점)
- [특정 버전을 직접 지정하고 싶을 때](#특정-버전을-직접-지정하고-싶을-때)
- [대안 — Gradle의 platform](#대안--gradle의-platform)
- [관련 노트](#관련-노트)

## 개요

Gradle에서 의존성 좌표는 그룹, 아티팩트, 버전 세 부분으로 이루어진다. 버전을 비워 두면 Gradle은 어떤 파일을 내려받을지 알 수 없다. 스프링 부트 프로젝트에서 버전을 비울 수 있는 것은 Gradle의 기본 동작이 아니라 플러그인이 끼어들어 채워 주는 결과다.

```kotlin
plugins {
    id("io.spring.dependency-management") version "1.1.7"
}

dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")   // 버전 없음
    compileOnly("org.projectlombok:lombok")                              // 버전 없음
}
```

## BOM이란

BOM(Bill of Materials)은 함께 써도 문제가 없다고 검증된 라이브러리 버전 목록이다. 스프링 부트는 릴리스마다 spring-boot-dependencies라는 이름의 BOM을 배포한다. 4.1.1용 BOM에는 스프링 프레임워크, 하이버네이트, 잭슨, 롬복, H2 등 수백 개 라이브러리의 버전이 적혀 있다.

io.spring.dependency-management 플러그인은 org.springframework.boot 플러그인과 함께 있을 때 현재 부트 버전에 맞는 BOM을 자동으로 가져온다. 그리고 dependencies 블록에서 버전이 빈 좌표를 만나면 BOM에서 해당 아티팩트의 버전을 찾아 채운다.

## 시나리오 — 플러그인을 빼고 빌드하면

플러그인이 실제로 무엇을 하는지는 빼 보면 드러난다. 아래 두 경우는 dependencies 블록이 같고 plugins 블록만 다르다.

### ❌ 플러그인 없이

```kotlin
plugins {
    java
    id("org.springframework.boot") version "4.1.1"
    // id("io.spring.dependency-management") version "1.1.7"
}

dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    compileOnly("org.projectlombok:lombok")
}
```

이 상태로 ./gradlew build를 실행하면 다음과 같이 실패한다. 아래 출력은 실제 실행 결과를 옮긴 것이다.

```text
FAILURE: Build failed with an exception.
> Could not resolve all files for configuration ':compileClasspath'.
   > Could not find org.projectlombok:lombok:.
   > Could not find org.springframework.boot:spring-boot-starter-web:.
```

좌표 끝에 콜론만 남고 버전이 비어 있다. Gradle은 org.projectlombok:lombok 뒤에 올 버전을 채워 줄 주체가 없어 빈 문자열 그대로 저장소를 조회했고, 그런 아티팩트는 존재하지 않는다.

### ✅ 플러그인과 함께

```kotlin
plugins {
    java
    id("org.springframework.boot") version "4.1.1"
    id("io.spring.dependency-management") version "1.1.7"
}

dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    compileOnly("org.projectlombok:lombok")
}
```

같은 dependencies 블록이 그대로 해석된다. 플러그인이 spring-boot-dependencies 4.1.1 BOM에서 lombok과 spring-boot-starter-web의 버전을 찾아 채운 뒤 Gradle에 넘기기 때문이다. 실제로 어떤 버전이 채워졌는지는 다음 명령으로 확인한다.

```bash
./gradlew dependencies --configuration compileClasspath
```

## 이점

부트 버전만 올리면 딸린 라이브러리 버전이 한꺼번에 맞춰진다. 4.1.1에서 다음 패치 버전으로 올릴 때 plugins 블록의 숫자 하나만 바꾸면 BOM이 바뀌고, 그 BOM에 적힌 수백 개 라이브러리 버전이 함께 따라온다.

서로 호환 범위가 있는 조합이 깨지지 않는다. 스프링 부트, 하이버네이트, 잭슨은 각각 독립적으로 릴리스되지만 특정 버전끼리만 함께 동작한다. 개발자가 세 라이브러리의 버전을 각자 고르면 그 조합이 검증된 적이 없을 수 있다. BOM에 적힌 조합은 스프링 팀이 함께 테스트한 것이다.

## 특정 버전을 직접 지정하고 싶을 때

특정 라이브러리만 BOM과 다른 버전을 써야 하면 그 좌표에 버전을 직접 적는다. 직접 적은 버전이 BOM보다 우선한다.

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")   // BOM이 채움
    implementation("com.fasterxml.jackson.core:jackson-databind:2.19.0") // 직접 지정이 우선
}
```

이 경우 BOM이 보장하던 호환성 검증에서 그 라이브러리 하나가 빠진다. 직접 지정한 버전이 나머지 조합과 맞는지는 개발자가 확인해야 한다.

## 대안 — Gradle의 platform

Gradle 자체에도 BOM을 읽는 기능이 있다. platform 함수로 BOM 좌표를 넘기면 플러그인 없이도 같은 동작을 한다.

```kotlin
plugins {
    java
    id("org.springframework.boot") version "4.1.1"
}

dependencies {
    implementation(platform("org.springframework.boot:spring-boot-dependencies:4.1.1"))
    implementation("org.springframework.boot:spring-boot-starter-web")
    compileOnly("org.projectlombok:lombok")
}
```

두 방식의 차이는 다음과 같다.

| | io.spring.dependency-management | platform |
|---|---|---|
| 부트 버전을 적는 곳 | plugins 블록 한 곳 | plugins 블록과 platform 좌표 두 곳 |
| 추가 플러그인 | 필요 | 불필요 |
| start.spring.io 기본값 | 이쪽을 생성 | 아님 |

platform 방식은 부트 버전을 두 군데 적게 되어 한쪽만 올리는 실수가 생길 수 있다. start.spring.io가 생성하는 기본 형태는 플러그인 쪽이므로, 특별한 이유가 없으면 생성된 형태를 그대로 쓴다.

## 관련 노트

- [[overview]] — 의존성 목록은 파일이 아니라 ./gradlew dependencies로 확인한다는 원칙
- [[plugins-vs-dependencies]] — 이 플러그인이 plugins 블록에 있으면서 dependencies 블록에 영향을 주는 이유
- [[dependency-configurations]] — 버전이 채워진 뒤 그 라이브러리가 어느 시점에 쓰이는지 정하는 구분
