---
title: Pydantic v2 Field Validators
type: pattern
tags: [pydantic, validation, python]
related:
  - "[[v1-vs-v2]]"
  - "[[language/python/typing/annotated]]"
last_reviewed: 2026-05-02
publish: false
---

# Pydantic v2 Field Validators

## 목차

- [기본 형태](#기본-형태)
- [모드 (mode) — 실행 시점 제어](#모드-mode--실행-시점-제어)
- [ValidationInfo — 외부 정보 접근](#validationinfo--외부-정보-접근)
- [context — 외부 정보 주입](#context--외부-정보-주입)
- [컨테이너 요소 검증 — `Annotated` 합성](#컨테이너-요소-검증--annotated-합성)
- [v1과의 주요 차이 정리](#v1과의-주요-차이-정리)
- [사고 모델](#사고-모델)
- [참고](#참고)

## 기본 형태

```python
from pydantic import BaseModel, field_validator

class User(BaseModel):
    name: str
    
    @field_validator('name')
    @classmethod
    def check(cls, v: str) -> str:
        if len(v) < 3:
            raise ValueError('too short')
        return v
```

- 데코레이터: **`@field_validator`** (v1의 `@validator`를 대체)
- `@classmethod` 명시 권장
- 타입 힌트 권장

## 모드 (mode) — 실행 시점 제어

```python
@field_validator('name', mode='before')   # 타입 검증/변환 전
@field_validator('name', mode='after')    # 타입 검증/변환 후 (기본값)
@field_validator('name', mode='wrap')     # 검증 흐름 자체를 가로챔
```

### before — Pydantic 표준 검증/변환 **이전**

- 입력은 **raw** (어떤 타입이든 들어올 수 있음)
- 입력 정규화, 타입 변환 가로채기에 적합

```python
@field_validator('amount', mode='before')
@classmethod
def normalize(cls, v):
    if isinstance(v, str):
        return int(v.replace(',', ''))  # "1,000" → 1000
    return v
```

### after — Pydantic 표준 검증/변환 **이후** (기본값)

- 입력은 **타입이 보장됨** (선언된 필드 타입)
- 비즈니스 규칙 검증에 적합

```python
@field_validator('amount', mode='after')
@classmethod
def check_range(cls, v: int) -> int:
    if v < 0:
        raise ValueError('must be positive')
    return v
```

### wrap — 검증 흐름 직접 제어

`handler`를 통해 표준 검증을 호출하거나 건너뛰거나 예외를 캐치할 수 있다.

```python
from pydantic import ValidatorFunctionWrapHandler

@field_validator('amount', mode='wrap')
@classmethod
def with_fallback(cls, v, handler: ValidatorFunctionWrapHandler):
    try:
        return handler(v)              # 표준 검증 실행
    except ValidationError:
        return 0                       # 실패 시 폴백
```

#### wrap 모드의 사고 모델

`wrap`은 **yield/middleware/AOP의 around 패턴**과 구조가 동일하다.

```
[before logic] → handler(v)  → [after logic]
                    ↑
              제어권 임시 이동
```

비유 가능한 패턴:
- `pytest fixture`의 `yield`
- ASGI middleware의 `await call_next(request)`
- 데코레이터의 내부 함수 호출
- Spring AOP의 `pjp.proceed()`

#### wrap이 필요한 경우

| 케이스 | 설명 |
|--------|------|
| **검증 실패 시 폴백** | `try/except` + 기본값 반환 |
| **검증 우회** | 특정 조건에서 `handler` 호출 안 함 |
| **여러 형식 시도** | 형식 변환 후 재검증 반복 |
| **에러 도메인 변환** | Pydantic 에러를 도메인 에러로 |

#### wrap이 불필요한 경우 (대부분)

before/after로 충분하면 그걸 쓴다. wrap은 **표현력은 강하지만 복잡도도 높다**.

## ValidationInfo — 외부 정보 접근

`info` 파라미터로 다른 필드 / 컨텍스트 / 메타데이터 접근.

```python
from pydantic import field_validator, ValidationInfo

class User(BaseModel):
    username: str
    name: str
    
    @field_validator('name')
    @classmethod
    def differ_from_username(cls, v: str, info: ValidationInfo) -> str:
        # 이미 검증된 다른 필드 접근
        if v == info.data.get('username'):
            raise ValueError('name cannot equal username')
        return v
```

`info`의 주요 속성:

| 속성 | 의미 |
|------|------|
| `info.data` | 이미 검증된 다른 필드들 (dict) |
| `info.field_name` | 현재 필드명 |
| `info.config` | 모델 config |
| `info.context` | 호출 시 주입된 외부 컨텍스트 (v2 신규) |

## context — 외부 정보 주입

`model_validate(data, context=...)`로 검증 시점에 외부 상태 주입.

```python
class CommentCreate(BaseModel):
    content: str
    
    @field_validator('content')
    @classmethod
    def check_permission(cls, v: str, info: ValidationInfo) -> str:
        user = (info.context or {}).get('current_user')
        if not user or not user.can_comment:
            raise ValueError('권한 없음')
        return v

# 호출
CommentCreate.model_validate(
    {"content": "..."},
    context={"current_user": current_user}
)
```

### 활용 케이스

| 외부 의존성 | context로 주입 권장? |
|------------|------------------|
| 현재 사용자 / 권한 | ✅ |
| 테넌트 / 정책 객체 | ✅ |
| 현재 시각 (테스트용) | ✅ |
| 피처 플래그 | ✅ |
| DB 세션 (유니크 검증 등) | ⚠️ 가능하지만 신중히 |
| 외부 API 호출 | ❌ 검증은 동기/순수해야 함 |

### 안전 룰

```python
# 1. 항상 디폴트 처리
ctx = info.context or {}
value = ctx.get('key', default)

# 2. 검증 외 비즈니스 로직 금지
@field_validator('email')
@classmethod
def check(cls, v, info):
    db = info.context['db']
    db.query(...)        # ✅ 읽기만
    # db.add(...)        # ❌ 부수효과 금지
    return v

# 3. 컨텍스트 의존성 명시적으로 문서화
"""
Required context:
    - current_user: User
    - tenant: Tenant
"""
```

## 컨테이너 요소 검증 — `Annotated` 합성

v1의 `each_item=True`는 v2에서 사라졌다. 대신 **요소 타입에 직접 검증 부착**.

```python
from typing import Annotated
from pydantic import AfterValidator

def upper(v: str) -> str:
    return v.upper()

UpperStr = Annotated[str, AfterValidator(upper)]

class Article(BaseModel):
    tags: list[UpperStr]  # 각 요소가 자동 대문자화
```

## v1과의 주요 차이 정리

| 항목 | v1 | v2 |
|------|----|----|
| 데코레이터 | `@validator` | `@field_validator` |
| 모드 | `pre=True/False` | `mode='before'/'after'/'wrap'` |
| 다른 필드 | 함수 인자 `values` | `info.data` |
| 외부 컨텍스트 | 없음 (글로벌/contextvars 우회) | `info.context` |
| 컨테이너 요소 | `each_item=True` | `Annotated` 합성 |
| 항상 실행 | `always=True` 옵션 | 기본 동작 |
| 재사용 | `allow_reuse=True` | `Annotated`로 자연스럽게 |
| `wrap` 모드 | 없음 | 신규 |

자세한 v1/v2 차이는 [[v1-vs-v2]] 참조.

## 사고 모델

> Validator는 두 단계로 분리된다:
>
> - **before** = "타입 만들기 전" — 입력 정규화
> - **after** = "타입 만들어진 후" — 비즈니스 규칙
> - **wrap** = "타입 만드는 과정 자체를 내가 운전" — 폴백/우회/변환

## 참고

- [Pydantic v2 docs — Validators](https://docs.pydantic.dev/latest/concepts/validators/)
- [Pydantic v2 docs — ValidationInfo](https://docs.pydantic.dev/latest/api/types/#pydantic.ValidationInfo)
