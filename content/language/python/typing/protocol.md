---
title: Protocol — 구조적 서브타이핑
type: concept
tags: [python, typing, structural-subtyping, oop]
related:
  - "[[annotated]]"
  - "[[architecture/clean-architecture]]"
last_reviewed: 2026-05-02
publish: false
---

# Protocol — 구조적 서브타이핑

## 목차

- [한 줄 정의](#한-줄-정의)
- [명목적 vs 구조적 서브타이핑](#명목적-vs-구조적-서브타이핑)
- [Protocol의 역할 (4가지)](#protocol의-역할-4가지)
- [ABC와의 비교](#abc와의-비교)
- [사용 패턴](#사용-패턴)
- [어떤 걸 언제 쓰나](#어떤-걸-언제-쓰나)
- [기본 권장 — 명시적 상속](#기본-권장--명시적-상속)
- [함정](#함정)
- [참고](#참고)

## 한 줄 정의

**"클래스가 어떤 모양(메서드/속성)이면 같은 타입으로 본다"**는 구조적 서브타이핑(Structural Subtyping)을 정적 타입 체커가 검증할 수 있게 만든 도구. (PEP 544, Python 3.8+)

## 명목적 vs 구조적 서브타이핑

### 명목적 (Nominal) — 전통 OOP, Java
"이 클래스가 부모 클래스를 **명시적으로 상속**해야 자식이다"

```python
from abc import ABC, abstractmethod

class Repository(ABC):
    @abstractmethod
    def find(self, id: str) -> Item: ...

class UserRepository(Repository):  # ← 상속 강제
    def find(self, id: str) -> Item: ...
```

### 구조적 (Structural) — Go interface, TypeScript, Python Protocol
"**모양만 같으면** 같은 타입으로 본다"

```python
from typing import Protocol

class Repository(Protocol):
    def find(self, id: str) -> Item: ...

class UserRepository:  # ← 상속 없음!
    def find(self, id: str) -> Item: ...

def get(repo: Repository, id: str) -> Item:
    return repo.find(id)

get(UserRepository(), "abc")  # ✅ 타입 체커 통과
```

> **Duck Typing의 정적 버전.** 런타임 duck typing은 Python의 오랜 특징이었고, Protocol은 거기에 **타입 체커 검증**을 더한 것.

## Protocol의 역할 (4가지)

### ① 인터페이스 명세

"이런 모양을 갖춘 객체"를 표현. 클래스 계층이 아니라 **계약(contract)**을 정의.

### ② 정적 타입 검사

mypy/pyright가 호환성을 검증 → 런타임 전에 "이 객체가 Protocol 만족 안 함" 발견.

### ③ 의존성 역전 메커니즘

도메인 코드가 인프라 코드를 모르게 만드는 도구. 인프라가 도메인의 Protocol을 import할 필요조차 없음.

### ④ 가벼운 문서화

"이 함수에 뭘 넘겨야 하는가"가 코드로 명시됨. ABC보다 가볍게 의도를 드러냄.

## ABC와의 비교

| 항목 | `ABC` | `Protocol` |
|------|-------|------------|
| 관계 결정 시점 | **선언 시** (상속) | **사용 시** (호환성 검사) |
| 상속 강제 | 필요 | 불필요 (선택적) |
| 미구현 시 | 인스턴스화 차단 | 타입 체커가 경고만 |
| 외부 클래스 적용 | 불가 | 가능 (수정 없이) |
| 런타임 검증 | 자동 | `@runtime_checkable` 필요 |

## 사용 패턴

### 1. 명시적 상속 — 의도 드러내기

```python
class Repository(Protocol):
    def find(self, id: str) -> Item: ...

class UserRepository(Repository):  # ← 상속 명시
    def find(self, id: str) -> Item: ...
```

**장점**: 어댑터 파일 자체에서 미구현 메서드 즉시 경고. "이건 Repository 구현체"가 한눈에 보임.

### 2. 구조적 매칭 — 외부 클래스 어댑팅

```python
class CacheStore(Protocol):
    def get(self, key: str) -> bytes | None: ...
    def set(self, key: str, value: bytes) -> None: ...

# 서드파티 라이브러리 (수정 불가)
import redis
client: CacheStore = redis.Redis()  # ✅ Redis가 우연히 같은 모양
```

### 3. `@runtime_checkable` — isinstance 가능

```python
from typing import Protocol, runtime_checkable

@runtime_checkable
class Closeable(Protocol):
    def close(self) -> None: ...

if isinstance(obj, Closeable):
    obj.close()
```

**주의**: 메서드 **이름만** 체크. 시그니처는 검사 안 함.

### 4. 어댑터가 어떤 메서드를 구현해야 하는지 판단하는 4가지 메커니즘

#### a. 사용처(call site)에서 타입 체커가 잡아냄
```python
def use(repo: Repository): ...
use(UserRepository())  # 미구현 시 여기서 mypy 에러
```

#### b. 명시적 상속 (가장 추천)
```python
class UserRepository(Repository):  # 어댑터 파일에서 즉시 피드백
    ...
```

#### c. 모듈 레벨 검증 (방어막)
```python
# infrastructure/user_repo.py
class UserRepository: ...

_: Repository = UserRepository()  # import 시 타입 체크 강제
```

#### d. 테스트로 호환성 확인
```python
def test_user_repo_satisfies_protocol():
    repo: Repository = UserRepository()
```

## 어떤 걸 언제 쓰나

| 상황 | 권장 |
|------|------|
| 도메인의 outbound port (Repository 등) | **Protocol** + 명시적 상속 |
| 도메인 내부 다형성 (전략 패턴 등) | **ABC** (도메인 안에선 명시적 의도가 좋음) |
| 외부 라이브러리 어댑팅 | **Protocol** (구조적 매칭) |
| 테스트용 fake/stub | **Protocol** (가벼움 우선, 상속 없이) |

## 기본 권장 — 명시적 상속

```python
class Repository(Protocol): ...

# 평소엔 이렇게 (의도 명시)
class UserRepository(Repository):
    def find(self, id: str) -> Item: ...

# 외부 클래스 래핑 시에만 구조적 매칭
class ThirdPartyAdapter:  # 상속 불가능한 외부 클래스
    def find(self, id: str) -> Item: ...
```

> 구조적 서브타이핑의 **유연성은 옵션으로 남기고**, 평소엔 명시적 의도를 드러내는 게 코드베이스 유지보수에 유리.

## 함정

### ① `@runtime_checkable`은 시그니처 검증 안 함

```python
@runtime_checkable
class Reader(Protocol):
    def read(self, n: int) -> bytes: ...

class WrongReader:
    def read(self):  # 시그니처 다름!
        return ""

isinstance(WrongReader(), Reader)  # True ❌ (메서드명만 체크)
```

→ 정적 타입 체커가 진짜 검증 도구.

### ② Protocol은 인스턴스화 안 됨

```python
class Repo(Protocol): ...
Repo()  # ❌ TypeError
```

ABC와 동일. 인터페이스 정의일 뿐 실체 아님.

### ③ ABC와 Protocol 둘 다 상속하면 충돌

```python
class A(ABC, Protocol): ...  # 가급적 피할 것 (의도 모호)
```

## 참고

- [PEP 544 — Protocols: Structural subtyping](https://peps.python.org/pep-0544/)
- [Python typing docs — Protocol](https://docs.python.org/3/library/typing.html#typing.Protocol)
