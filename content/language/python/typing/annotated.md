---
title: Annotated
type: concept
tags: [python, typing, metadata]
related:
  - "[[protocol]]"
  - "[[framework/fastapi/dependency-injection]]"
  - "[[framework/pydantic/validators]]"
last_reviewed: 2026-05-02
publish: false
---

# Annotated

## 목차

- [한 줄 정의](#한-줄-정의)
- [등장 배경](#등장-배경)
- [핵심 동작](#핵심-동작)
- [Annotated와 DI의 관계](#annotated와-di의-관계)
- [사용 패턴](#사용-패턴)
- [함정 / 자주 하는 오해](#함정--자주-하는-오해)
- [사고 모델](#사고-모델)
- [참고](#참고)

## 한 줄 정의

타입에 **메타데이터를 부착**할 수 있게 해주는 typing 도구. (Python 3.9+, `typing.Annotated`)

> ⚠️ **자주 하는 오해**: "`Annotated`는 DI 도구다" — **아니다.**
> `Annotated`는 **순수 typing 기능**이다. DI, 검증, 문서화 등 **무엇이든** 메타데이터로 부착할 수 있는 **범용 컨테이너**일 뿐. 자세히는 아래 [Annotated와 DI의 관계](#annotated와-di의-관계) 참조.

## 등장 배경

기존 Python 타입 힌트는 "이 값은 무엇이다"만 표현 가능했다. 하지만 실무에선:

- "이 값은 `int`인데, **0보다 커야 함**"
- "이 값은 `Session`인데, **DI로 주입돼야 함**"
- "이 값은 `str`인데, **trim 처리되어야 함**"

같은 **부가 정보**가 필요했다. `Annotated`는 이를 **타입과 함께 들고 다닐 수 있는 컨테이너**를 제공한다.

## 핵심 동작

```python
from typing import Annotated

# 형식: Annotated[실제_타입, 메타데이터1, 메타데이터2, ...]
UserId = Annotated[int, "user identifier"]
PositiveInt = Annotated[int, Field(gt=0)]
```

- 타입 체커는 **첫 번째 인자(실제 타입)만** 본다
- 라이브러리(FastAPI, Pydantic 등)는 **메타데이터를 읽어** 자체 동작 수행
- **여러 메타데이터를 동시에** 부착 가능

## Annotated와 DI의 관계

### 결론

**`Annotated` 자체는 DI와 무관하다.** DI는 `Annotated` 위에 얹는 **하나의 활용 사례**일 뿐이다.

### 레이어 구분

| 레이어 | 정체 |
|--------|------|
| **`Annotated`** | Python 표준 typing 도구. 메타데이터 컨테이너. **DI를 모름** |
| **DI (Dependency Injection)** | 패턴/사상. 언어/라이브러리 무관 |
| **`Depends(...)`** | FastAPI가 정의한 **DI 마커 객체** |
| **`Annotated[T, Depends(...)]`** | FastAPI가 `Annotated`를 **운반 수단으로 빌려서** DI를 표현한 것 |

### `Annotated`의 다양한 용도 (DI는 그중 하나)

```python
from typing import Annotated

# ① 단순 문서화 — DI 무관
Meters = Annotated[float, "단위: 미터"]

# ② Pydantic 검증 — DI 무관
PositiveInt = Annotated[int, Field(gt=0)]

# ③ Pydantic 변환 함수 — DI 무관
TrimStr = Annotated[str, AfterValidator(str.strip)]

# ④ Typer/Click CLI 인자 — DI 무관
def cli(name: Annotated[str, typer.Option("--name")]): ...

# ⑤ FastAPI Query/Path/Header — DI 무관 (검증/파싱)
SearchQ = Annotated[str, Query(min_length=3)]

# ⑥ FastAPI Depends — 비로소 DI
DBSession = Annotated[Session, Depends(get_db)]
```

→ **①~⑤는 DI가 아니다.** ⑥만 DI.

### 사고 모델

```
Annotated         = "메타데이터 운반 트럭"
Depends/Field/Query = "트럭에 실리는 화물"
   - Depends → 화물 종류가 "DI"
   - Field → 화물 종류가 "검증/문서"
   - Query → 화물 종류가 "쿼리 파라미터 파싱"
```

`Annotated`라는 도구를 익히면 **여러 라이브러리에서 동일하게 적용**할 수 있다. FastAPI/Pydantic/Typer 모두 이 트럭에 자기 화물을 싣는다.

자세한 DI 활용은 [[framework/fastapi/dependency-injection]] 참조.

## 사용 패턴

### 1. 기본 타입에 별칭 + 메타데이터

```python
from typing import Annotated
from pydantic import Field

UserId = Annotated[int, Field(gt=0, description="User ID")]
Email = Annotated[str, Field(pattern=r"^[\w.+-]+@[\w-]+\.[\w.-]+$")]

class User(BaseModel):
    id: UserId
    email: Email
```

### 2. DI 메타데이터 (FastAPI)

```python
from typing import Annotated
from fastapi import Depends

DBSession = Annotated[Session, Depends(get_db)]

def get_user(db: DBSession): ...
```

자세한 내용은 [[framework/fastapi/dependency-injection]] 참조.

### 3. 검증 함수 부착 (Pydantic v2)

```python
from typing import Annotated
from pydantic import AfterValidator

def trim(v: str) -> str:
    return v.strip()

CleanStr = Annotated[str, AfterValidator(trim)]

class Article(BaseModel):
    title: CleanStr  # 자동으로 strip 적용
```

### 4. 메타데이터 합성

```python
SearchQuery = Annotated[
    str,
    Query(min_length=3, max_length=50),  # FastAPI 검증
    Field(description="검색어"),           # Pydantic 메타데이터
]
```

여러 메타데이터를 한 타입에 동시 부착 가능. 각 라이브러리가 자기에게 필요한 것만 읽는다.

## 함정 / 자주 하는 오해

### ① "Annotated가 없어도 동작하는데 왜 쓰나?"

옛날 스타일 (FastAPI 예시):

```python
def get_user(db: Session = Depends(get_db)): ...
```

문제점:
- `db`가 옵셔널 파라미터처럼 보이지만 실제론 필수
- 직접 호출 시 `db`에 `Depends` 객체가 들어감 → 오작동
- 타입 체커가 "기본값 타입(`Depends`)과 선언 타입(`Session`)이 다르다"며 경고

`Annotated`는 **타입과 메타데이터를 한 자리에서 분리** → 시그니처가 솔직해짐.

### ② "Annotated는 런타임에 영향이 없다"

타입 체커 입장에선 무시되지만, **`get_type_hints(include_extras=True)`로 메타데이터 추출 가능**. 라이브러리가 이를 활용해 동작을 결정한다.

```python
from typing import get_type_hints

hints = get_type_hints(get_user, include_extras=True)
# {'db': Annotated[Session, Depends(get_db)]}
```

### ③ "여러 라이브러리 메타데이터를 섞어도 되나?"

된다. 각 라이브러리는 자기 메타데이터만 인식하고 나머진 무시한다.

```python
# Pydantic, FastAPI 메타데이터 공존
UserAge = Annotated[
    int,
    Field(ge=0, le=150),    # Pydantic
    Query(description="..."), # FastAPI Query 파라미터로 쓸 때
]
```

## 사고 모델

> **타입 자체는 깔끔하게 유지하고, 부가 정보는 어노테이션으로 주렁주렁 매단다.**
>
> 함수 시그니처에서 `=` 우측 자리(기본값)는 본래 용도(진짜 기본값)에만 사용하고, **DI/검증/문서 같은 메타데이터는 타입 쪽으로 이동**시키는 것이 핵심.

## 참고

- [PEP 593 — Flexible function and variable annotations](https://peps.python.org/pep-0593/)
- [Python typing docs — Annotated](https://docs.python.org/3/library/typing.html#typing.Annotated)
