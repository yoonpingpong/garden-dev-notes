---
title: FastAPI Dependency Injection
type: pattern
tags: [fastapi, di, python]
prerequisites:
  - "[[language/python/typing/annotated]]"
related:
  - "[[language/python/typing/protocol]]"
last_reviewed: 2026-05-03
publish: false
---

# FastAPI Dependency Injection

## 목차

- [핵심 아이디어](#핵심-아이디어)
- [FastAPI의 동작 방식 (내부)](#fastapi의-동작-방식-내부)
- [의존성 생성 시점](#의존성-생성-시점)
- [의존성 별칭 모음 (`deps.py` 패턴)](#의존성-별칭-모음-depspy-패턴)
- [의존성 체이닝](#의존성-체이닝)
- [yield 기반 의존성 (리소스 관리)](#yield-기반-의존성-리소스-관리)
- [의존성 캐싱 (같은 요청 내)](#의존성-캐싱-같은-요청-내)
- [싱글톤이 필요할 땐 — `lifespan`](#싱글톤이-필요할-땐--lifespan)
- [테스트 — `dependency_overrides`](#테스트--dependency_overrides)
- [함정](#함정)
- [사고 모델](#사고-모델)
- [참고](#참고)

## 핵심 아이디어

요청 처리 시점에 필요한 객체(DB 세션, 인증된 사용자, 캐시 클라이언트 등)를 **파라미터로 선언만 하면 FastAPI가 자동 주입**해준다.

```python
DBSession = Annotated[Session, Depends(get_db)]

@app.get("/users/{id}")
def get_user(id: int, db: DBSession):
    return db.query(User).filter_by(id=id).first()
```

핸들러는 **"무엇이 필요한가"** 만 선언하고, **"어떻게 얻는가"** 는 의존성 함수(`get_db`)가 책임진다. 이로써 핸들러는 자기 책임에만 집중하고, 외부 자원 획득 방식은 분리된다.

## FastAPI의 동작 방식 (내부)

> 아래 코드는 **실제 FastAPI 소스가 아닌 의사 코드**다. 흐름 이해용으로만 활용.
> 실제 구현은 `fastapi/dependencies/utils.py`의 `solve_dependencies()` 참조.

FastAPI의 의존성 관리는 **2단계로 분리**되어 있다.

| 단계 | 시점 | 결과물 |
|------|------|--------|
| ① **분석** (introspection) | 앱 시작 시 **1번** | `Dependant` 객체 (메타데이터/설계도) |
| ② **실행** (execution) | **매 요청마다** | 실제 객체 (Session, User 등) |

> 핵심: **"분석은 한 번, 실행은 매번"**

### ① 앱 시작 시 — 분석 (1번만)

라우트 등록 시점에 핸들러의 시그니처를 분석해 의존성 트리를 미리 만들어둔다.

```python
# 의사 코드 — 앱 시작 시 1번
def register_route(endpoint):
    sig = inspect.signature(endpoint)
    dependant = build_dependant_tree(sig)   # 메타데이터 트리
    routes[endpoint] = dependant            # 캐시
```

이 시점에 **일어나는 일**:
- `Annotated` 메타데이터에서 `Depends` 마커 추출
- "어떤 의존성 함수를 호출해야 하는가"를 트리(`Dependant`)로 정리
- 하위 의존성(체이닝)도 모두 미리 해소

이 시점에 **일어나지 않는 일**:
- 의존성 함수 호출 (`get_db()` 등)
- 실제 객체 생성 (Session, User 인스턴스 등)

### ② 매 요청마다 — 실행

미리 만들어둔 `Dependant` 트리를 참조해 의존성 함수를 호출하고 결과를 핸들러에 주입한다.

```python
# 의사 코드 — 매 요청마다
def handle_request(endpoint, request):
    dependant = routes[endpoint]              # ← 캐시된 트리 조회 (분석 X)

    kwargs = {}
    for sub in dependant.dependencies:
        kwargs[sub.name] = call(sub.function) # ← 의존성 실행

    return endpoint(**kwargs)                 # ← 명시적 인자로 핸들러 호출
```

이 시점에 일어나는 일:
- 캐시된 `Dependant` 트리 조회 (가벼운 작업)
- 의존성 함수 호출 → 실제 객체 생성
- 핸들러에 명시적 키워드 인자로 주입

### 왜 2단계로 나눴나

`inspect.signature()`는 무거운 introspection 작업. 매 요청마다 수행하면 성능 손실이 크다.

그래서 FastAPI는:
- **앱 시작 시 1번** — 무거운 분석을 끝내고 결과 캐시
- **매 요청 시** — 캐시 조회 + 의존성 함수 실행 (가벼움)

이 분리가 FastAPI의 빠른 요청 처리 비결 중 하나.

### 비유 — 레스토랑

```
[레스토랑 오픈 = 앱 시작]
 └─ 레시피 카드 작성 (1번)
    "파스타 = 면 + 토마토 + 바질 필요"
    ※ 면 안 삶음, 토마토 안 썲

[손님 #1 주문 = 요청 #1]
 └─ 레시피 카드 보고 → 면 삶고 토마토 썰어 → 새 접시 제공

[손님 #2 주문 = 요청 #2]
 └─ 같은 레시피 카드 → 또 새로 삶고 썰어 → 또 새 접시 제공
```

- **레시피 카드** = `Dependant` 트리 (앱 시작 시 1번)
- **실제 파스타 접시** = 의존성 객체 (매 요청마다 새로)

## 의존성 생성 시점

> ⚠️ **자주 하는 오해**: "서버 시작 시 의존성 객체를 미리 만들어두고, 요청 시 그 싱글톤을 주입한다"
> — **틀림.** 매 요청마다 의존성 함수가 새로 호출된다.

### 시점별 동작

```
[서버 시작]
  └─ FastAPI가 핸들러 시그니처 분석 (메타데이터만 읽음)
  └─ Depends(get_db) 마커 발견 → "요청 오면 호출하자"고 기록
  └─ get_db()는 호출되지 않음

[요청 #1 도착]
  ├─ get_db() 호출 → 새 Session 생성
  ├─ 핸들러에 주입
  ├─ 응답 반환
  └─ get_db()의 finally → db.close()

[요청 #2 도착]
  ├─ get_db() 다시 호출 → 또 새 Session 생성
  └─ ...

→ 요청 #1과 #2의 db는 완전히 다른 객체
```

### 검증 코드

```python
def get_db():
    print("get_db called!")
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

서버 시작 시 출력: (없음)
요청 3번 보냈을 때:
```
get_db called!
get_db called!
get_db called!
```

### 왜 이렇게 설계됐나

| 이유 | 설명 |
|------|------|
| **요청 격리** | DB 세션·트랜잭션·사용자 컨텍스트는 요청마다 달라야 함. 공유 시 트랜잭션 꼬임/보안 사고 |
| **동시성 안전** | 매 요청이 새 객체 → 공유 가변 상태 문제 없음 |
| **테스트 친화** | 매번 새로 생성 → 테스트 간 상태 누수 없음 |

### Spring/Java와의 차이 (사고 모델 전환)

| 측면 | Spring | FastAPI |
|------|--------|---------|
| 기본 스코프 | **싱글톤** (`@Bean`) | **요청 단위** (per-request) |
| 객체 생성 시점 | 서버 시작 시 | 요청 도착 시 |
| 다른 스코프 지정 | `@RequestScope`, `@Prototype` 명시 | 기본이 이미 요청 단위 |
| 싱글톤 필요 시 | 기본값 | 명시적 처리 (`lifespan` 사용) |

> **Spring**: "기본이 싱글톤, 요청 단위로 하려면 명시"
> **FastAPI**: "기본이 요청 단위, 싱글톤으로 하려면 명시"
>
> 디폴트가 정반대.

## 의존성 별칭 모음 (`deps.py` 패턴)

대규모 프로젝트의 표준 구조. 의존성을 한 곳에 정의하고 라우터에서 import.

```python
# app/deps.py
DBSession = Annotated[Session, Depends(get_db)]
CurrentUser = Annotated[User, Depends(get_current_user)]
AdminUser = Annotated[User, Depends(verify_admin)]
RedisClient = Annotated[Redis, Depends(get_redis)]
Settings = Annotated[AppSettings, Depends(get_settings)]
```

```python
# app/routers/users.py
from app.deps import DBSession, CurrentUser

@router.get("/me")
def me(user: CurrentUser, db: DBSession):
    ...
```

장점:
- 의존성 정의가 **한 곳**에 모임 → 변경 시 한 곳만 수정
- 별칭 이름이 **의도를 드러냄** (`CurrentUser`가 "현재 사용자"임이 즉시 보임)

## 의존성 체이닝

의존성 함수 안에서 또 다른 의존성을 주입받을 수 있다.

```python
def get_token(authorization: Annotated[str, Header()]) -> str:
    if not authorization.startswith("Bearer "):
        raise HTTPException(401)
    return authorization[7:]

def get_current_user(
    token: Annotated[str, Depends(get_token)],
    db: DBSession,
) -> User:
    return db.query(User).filter_by(token=token).first()

CurrentUser = Annotated[User, Depends(get_current_user)]
```

→ 의존성 그래프가 자동으로 해소됨. `CurrentUser` 요청 시 `get_token` → `get_current_user` 순으로 실행.

## yield 기반 의존성 (리소스 관리)

리소스 setup/teardown을 한 함수에 묶을 수 있다.

```python
def get_db():
    db = SessionLocal()
    try:
        yield db          # ← 핸들러로 제어 넘김
    finally:
        db.close()        # ← 응답 후 자동 정리
```

- `yield` 전: setup
- `yield` 후 (`finally`): teardown
- 핸들러에서 예외 발생해도 정리 보장

## 의존성 캐싱 (같은 요청 내)

기본 동작: **같은 요청 안에서 동일 의존성은 한 번만 호출**되고 결과 재사용.

```python
def get_settings() -> AppSettings: ...

Settings = Annotated[AppSettings, Depends(get_settings)]

def handler_a(s: Settings): ...
def handler_b(s: Settings): ...

# 한 요청에서 handler_a, handler_b가 둘 다 Settings 사용해도
# get_settings는 한 번만 호출됨
```

캐시 비활성화: `Depends(get_settings, use_cache=False)`

## 싱글톤이 필요할 땐 — `lifespan`

DB **연결 풀**, ML 모델, HTTP 클라이언트처럼 **생성 비용이 큰 객체**는 매 요청마다 새로 만들면 안 된다. 서버 시작 시 한 번만 생성하고 모든 요청이 공유해야 한다.

이때 사용하는 것이 `lifespan` context manager.

### 패턴

```python
from contextlib import asynccontextmanager

@asynccontextmanager
async def lifespan(app: FastAPI):
    # 서버 시작 시 한 번 — 진짜 싱글톤
    app.state.db_pool = create_pool(DATABASE_URL)
    app.state.ml_model = load_model()
    app.state.http_client = httpx.AsyncClient()

    yield  # 서버 가동 중

    # 서버 종료 시 한 번 — 정리
    await app.state.db_pool.close()
    await app.state.http_client.aclose()

app = FastAPI(lifespan=lifespan)
```

의존성 함수가 풀에서 **요청별 객체**를 꺼내 제공:

```python
def get_db(request: Request):
    pool = request.app.state.db_pool   # ← 싱글톤 풀 (서버 시작 시 생성됨)
    db = pool.acquire()                # ← 매 요청마다 세션은 새로 발급
    try:
        yield db
    finally:
        pool.release(db)
```

### 두 계층의 분리

```
[lifespan 계층 = 진짜 싱글톤]
  ├─ db_pool         (서버 시작 시 1번 생성)
  ├─ ml_model        (서버 시작 시 1번 로드)
  └─ http_client     (서버 시작 시 1번 생성)
        ↓
[Depends 계층 = 요청 단위]
  ├─ get_db()        매 요청마다 풀에서 세션 발급/반환
  ├─ get_predictor() 매 요청마다 모델 참조 + 입력 처리
  └─ get_api_call()  매 요청마다 클라이언트로 호출
```

**핵심**: 비싼 자원 자체는 `lifespan`, 그 자원으로 만든 **요청별 객체**는 `Depends`.

### 자주 만나는 함정

#### ❌ 의존성 함수에서 비싼 객체 매번 생성

```python
def get_ml_predictor():
    model = load_model_from_disk()  # ❌ 매 요청마다 디스크에서 로드!
    return Predictor(model)
```

매 요청마다 모델 로딩 비용이 발생해 서버가 느려지거나 죽는다.

#### ✅ lifespan에서 한 번 + Depends로 꺼내쓰기

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    app.state.model = load_model_from_disk()  # 1번
    yield

def get_predictor(request: Request) -> Predictor:
    return Predictor(request.app.state.model)  # 매 요청 — 가벼움
```

## 테스트 — `dependency_overrides`

테스트 시 의존성을 가짜 구현으로 교체.

```python
from fastapi.testclient import TestClient

def fake_db():
    return FakeSession()

app.dependency_overrides[get_db] = fake_db

client = TestClient(app)
response = client.get("/users/1")
```

테스트 종료 후: `app.dependency_overrides.clear()`

→ 실제 DB/외부 시스템 없이도 핸들러 검증 가능.

## 함정

### ① 의존성 함수가 너무 많은 일을 함

의존성 함수는 **객체 제공**만 책임진다. 비즈니스 로직, 부수효과는 다른 계층으로.

```python
# ❌ DI 함수에 비즈니스 로직 / 부수효과
def get_current_user(...) -> User:
    user = db.query(...)
    user.last_login = now()
    db.commit()           # ← 부수효과
    return user

# ✅ 조회만. 비즈니스는 핸들러/서비스로
def get_current_user(...) -> User:
    return db.query(...)
```

### ② 순환 의존성

A → B → A 같은 그래프는 FastAPI가 해소하지 못한다. 의존성 그래프는 **DAG(비순환)** 이어야 함.

### ③ 의존성 캐시를 전역 싱글톤으로 오해

요청 단위 캐시일 뿐, **요청 간엔 캐시되지 않는다**. 전역 싱글톤이 필요하면 [싱글톤이 필요할 땐 — `lifespan`](#싱글톤이-필요할-땐--lifespan) 참조.

## 사고 모델

> FastAPI DI는 **"이 핸들러는 이런 의존성을 필요로 한다"는 선언**.
>
> 핸들러는 **무엇이 필요한지만** 명시하고, **어떻게 얻는지는** 의존성 함수에 위임한다. 이 분리가 테스트 친화성·재사용성·역할 분리를 만든다.

## 참고

- [FastAPI 공식 문서 — Dependencies](https://fastapi.tiangolo.com/tutorial/dependencies/)
- [FastAPI 소스 — `fastapi/dependencies/utils.py`](https://github.com/fastapi/fastapi/blob/master/fastapi/dependencies/utils.py) — `solve_dependencies()` 실제 구현
- [FastAPI 소스 — `fastapi/dependencies/models.py`](https://github.com/fastapi/fastapi/blob/master/fastapi/dependencies/models.py) — `Dependant` 클래스
