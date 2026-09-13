---
title: Gradle 개요
type: concept
tags: [gradle, build-tool, jvm, java, kotlin]
related:
  - "[[plugins-vs-dependencies]]"
  - "[[bom-version-management]]"
  - "[[dependency-configurations]]"
last_reviewed: 2026-09-13
publish: true
---

# Gradle 개요

Gradle 빌드 스크립트는 읽히는 설정 파일이 아니라 실행되는 프로그램이다. 이 노트는 그 차이가 무엇을 바꾸는지, 그리고 JVM 진영에 갓 들어온 개발자가 어디까지 알아야 하는지를 정리한다.

## 목차

- [개요](#개요)
- [선언형 설정과 실행형 스크립트](#선언형-설정과-실행형-스크립트)
- [빌드를 구성하는 파일](#빌드를-구성하는-파일)
- [Gradle Wrapper](#gradle-wrapper)
- [태스크](#태스크)
- [어디까지 알아야 하나](#어디까지-알아야-하나)
- [관련 노트](#관련-노트)

## 개요

Gradle을 처음 만나면 build.gradle.kts를 pyproject.toml이나 package.json 같은 설정 파일로 읽게 된다. 이 오해가 초기 혼란의 대부분을 만든다. build.gradle.kts는 Kotlin 소스 파일이고, Gradle은 이 파일을 컴파일해서 실행한다. 파일 안의 dependencies { ... }는 데이터 구조가 아니라 함수 호출이다.

이 사실을 받아들이면 그 뒤의 규칙들이 따라온다. 스크립트에 if 문을 쓸 수 있고, 반복문을 돌릴 수 있고, 그래서 실행 시점이 언제인지가 문제가 되기 시작한다. 실행 시점 문제는 별도 노트에서 다룬다.

## 선언형 설정과 실행형 스크립트

Python의 pyproject.toml은 선언형이다. TOML 파서가 읽어서 자료구조로 만들고, 그 자료구조를 poetry나 pip이 해석한다. 파일 자체는 아무 일도 하지 않는다. 그래서 순서가 결과를 바꾸지 않고, 파일만 보면 최종 상태를 알 수 있다.

```toml
# pyproject.toml — 데이터. 파서가 읽을 뿐이다.
[project]
dependencies = ["httpx>=0.27", "pydantic>=2.7"]
```

Gradle의 build.gradle.kts는 실행형이다. Gradle이 이 파일을 하나의 Kotlin 클래스로 컴파일한 뒤 실행하고, 실행 과정에서 호출된 함수들이 빌드 모델을 조립한다.

```kotlin
// build.gradle.kts — 코드. 위에서 아래로 실행된다.
dependencies {
    implementation("io.ktor:ktor-client-core:3.0.0")
}
```

여기서 dependencies는 블록 문법처럼 보이지만 람다를 인자로 받는 함수이고, implementation("...")도 함수 호출이다. 그래서 다음 같은 코드가 문법적으로 성립한다.

```kotlin
dependencies {
    listOf("core", "cio", "logging").forEach { module ->
        implementation("io.ktor:ktor-client-$module:3.0.0")
    }
}
```

pyproject.toml에서는 표현할 수 없는 형태다. 이 유연성이 Gradle의 힘이자, 남이 짠 빌드 스크립트를 읽기 어렵게 만드는 원인이다.

실무에서 이 차이가 드러나는 지점은 이렇다. 의존성 목록을 알고 싶을 때 pyproject.toml은 파일을 열면 끝나지만, build.gradle.kts는 파일을 읽는 것만으로 부족할 때가 있다. 플러그인이 의존성을 추가하기도 하고 버전을 대신 정하기도 하기 때문이다. 그래서 실제 결과는 파일이 아니라 명령으로 확인한다.

```bash
./gradlew dependencies
```

## 빌드를 구성하는 파일

Gradle 프로젝트 루트에 항상 있는 것들이다.

| 파일 | 역할 |
|---|---|
| settings.gradle.kts | 어떤 모듈이 이 빌드에 속하는지 선언. 루트에 하나만 존재 |
| build.gradle.kts | 각 모듈이 무엇을 하는지 기술. 모듈마다 하나씩 |
| gradle/wrapper/gradle-wrapper.properties | 이 프로젝트가 쓸 Gradle 버전 고정 |
| gradlew, gradlew.bat | Wrapper 실행 스크립트 |

settings.gradle.kts가 없으면 Gradle은 그 디렉터리를 멀티모듈 빌드의 루트로 인식하지 못한다. 모듈을 추가했는데 IDE가 인식하지 못한다면 이 파일에 등록을 빠뜨린 경우가 대부분이다.

```kotlin
// settings.gradle.kts
rootProject.name = "shop"

include(":shop-api")
include(":shop-domain")
```

## Gradle Wrapper

Wrapper는 프로젝트가 쓸 Gradle 버전을 저장소 안에 고정하는 장치다. gradlew를 실행하면 gradle-wrapper.properties에 적힌 버전을 내려받아 그 버전으로 빌드한다. 로컬에 Gradle이 설치되어 있지 않아도 동작한다.

```properties
# gradle/wrapper/gradle-wrapper.properties
distributionUrl=https\://services.gradle.org/distributions/gradle-9.0-bin.zip
```

규칙은 하나다. 항상 ./gradlew를 쓰고, 로컬에 설치한 gradle 명령은 쓰지 않는다.

```bash
./gradlew build     # 프로젝트가 지정한 버전으로 빌드
gradle build        # 내 로컬에 깔린 버전으로 빌드 — 팀원과 결과가 달라질 수 있다
```

이 규칙을 어기면 실패는 조용히 온다. 로컬에 Gradle 8.5가 깔려 있고 프로젝트가 9.0을 요구할 때, 대부분의 빌드는 그냥 성공하다가 특정 플러그인이나 문법에서만 깨진다. 원인을 찾기 어려운 부류의 실패다.

Gradle 버전을 올릴 때도 로컬 설치본을 건드리지 않고 Wrapper를 갱신한다.

```bash
./gradlew wrapper --gradle-version 9.0
```

## 태스크

Gradle에서 실행 단위는 태스크다. build나 test 같은 이름은 명령어가 아니라 태스크 이름이며, 플러그인이 등록한다. 예를 들어 test 태스크는 Gradle 자체가 아니라 java 플러그인이 만든다. bootRun은 Spring Boot 플러그인이 만든다. 그래서 프로젝트마다 쓸 수 있는 태스크가 다르다.

현재 프로젝트가 가진 태스크는 명령으로 확인한다.

```bash
./gradlew tasks
```

자주 쓰는 것들이다.

| 태스크 | 하는 일 |
|---|---|
| clean | build 디렉터리 삭제 |
| compileJava, compileKotlin | 소스 컴파일 |
| test | 테스트 실행 |
| build | 컴파일 + 테스트 + 패키징 |
| bootRun | 애플리케이션 실행 (Spring Boot 플러그인) |
| bootJar | 실행 가능한 jar 생성 (Spring Boot 플러그인) |

태스크 사이에는 의존 관계가 있고, Gradle이 이를 계산해 필요한 것만 순서대로 실행한다. ./gradlew build를 실행하면 컴파일과 테스트가 먼저 도는 이유가 이것이다. 반대로 이미 최신 상태인 태스크는 건너뛰고 UP-TO-DATE로 표시한다.

## 어디까지 알아야 하나

Gradle은 끝까지 파고들면 빌드 엔지니어링이라는 별도 분야가 된다. 애플리케이션 개발자에게 필요한 선은 다음과 같다.

1단계 — 남의 프로젝트에 들어가 일할 수 있는 수준

- 위에서 다룬 파일 구조, Wrapper, 태스크
- 의존성 configuration의 차이 (implementation, api, runtimeOnly, compileOnly 등)
- 버전이 적혀 있지 않은 의존성이 왜 동작하는지 (BOM과 플러그인의 버전 관리)
- 문제 진단 명령 세 개: dependencies, dependencyInsight, --stacktrace

2단계 — 팀의 빌드를 고치고 개선할 수 있는 수준

- 멀티모듈 구성과 모듈 간 의존
- 버전 카탈로그 (gradle/libs.versions.toml)
- 컨벤션 플러그인 (buildSrc 또는 build-logic)
- 빌드 페이즈 (initialization, configuration, execution)
- 태스크 커스터마이징과 소스셋 분리

3단계 — 필요해질 때 찾아보면 되는 영역

- 커스텀 플러그인을 별도 아티팩트로 배포
- 태스크 입출력 선언과 증분 빌드, 빌드 캐시와 설정 캐시 튜닝
- Artifact transform, capability, variant-aware 의존성 해석

목표는 Gradle을 마스터하는 것이 아니라, 빌드가 깨졌을 때 원인을 스스로 찾을 수 있는 상태다. 그 지점을 가르는 것은 대부분 의존성 configuration에 대한 이해와 dependencyInsight 사용법이다.

## 관련 노트

작성된 노트:

- [[plugins-vs-dependencies]] — plugins는 빌드 도구를 확장하고 dependencies는 애플리케이션이 쓸 라이브러리를 지정한다
- [[bom-version-management]] — 버전이 비어 있는 의존성이 왜 동작하는가. io.spring.dependency-management 플러그인과 BOM
- [[dependency-configurations]] — implementation, compileOnly, runtimeOnly, testImplementation, annotationProcessor의 구분. 멀티모듈에서 implementation과 api의 차이

아래 노트는 아직 작성 전이며, 만들 때 이 노트와 양방향으로 연결한다.

- build-phases — 빌드 페이즈. 스크립트가 언제 실행되는가
- dependency-resolution — 버전 충돌 해결 방식, dependencyInsight
- version-catalog — libs.versions.toml로 버전을 한곳에서 관리하기
- multi-module — 모듈 분리와 컨벤션 플러그인
