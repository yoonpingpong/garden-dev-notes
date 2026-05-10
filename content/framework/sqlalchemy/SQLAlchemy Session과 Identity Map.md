---
title: SQLAlchemy Session과 Identity Map
type: concept
tags: [sqlalchemy, orm, async, jpa-comparison]
last_reviewed: 2026-05-10
publish: true
---

# SQLAlchemy Session과 Identity Map

SQLAlchemy의 `Session` (`AsyncSession`)이 JPA의 `EntityManager` 역할을 그대로 수행한다. Identity Map은 JPA의 영속성 컨텍스트(1차 캐시)와 동일한 메커니즘으로, 같은 세션 안에서 동일한 row가 항상 같은 파이썬 객체로 매핑되도록 보장한다.

## 목차

- [JPA와의 대응](#jpa와의-대응)
- [session.get — PK 단건 조회](#sessionget--pk-단건-조회)
- [일반 select와 Identity Map의 상호작용](#일반-select와-identity-map의-상호작용)
- [강제 갱신 — refresh, expire, populate_existing](#강제-갱신--refresh-expire-populate_existing)
- [정리](#정리)

## JPA와의 대응

| JPA | SQLAlchemy |
|-----|------------|
| `EntityManager` | `Session` / `AsyncSession` |
| 영속성 컨텍스트 (1차 캐시) | Identity Map |
| `@Id` 기반 캐시 키 | `(mapper, primary_key)` 튜플 |
| `em.find(Entity, id)` | `session.get(Entity, id)` |
| Persistent / Detached / Transient | Persistent / Detached / Transient (용어 동일) |
| flush / dirty checking | `session.flush()` / autoflush + change tracking |
| `em.refresh(entity)` | `session.refresh(obj)` |
| `em.clear()` | `session.expire_all()` |

## session.get — PK 단건 조회

```python
row = await session.get(UserModel, user_id)
```

동작:

1. **Identity Map 우선 조회** — 같은 세션 안에 이미 로드된 인스턴스가 있으면 SQL을 발행하지 않고 그것을 반환.
2. **없으면 단건 SELECT** — `WHERE pk = :id` 한 행만 조회.
3. **반환값** — 매칭 row가 없으면 `None`.

```python
a1 = await session.get(UserModel, some_id)  # SQL 발행
a2 = await session.get(UserModel, some_id)  # SQL 없음, identity map hit

assert a1 is a2  # True
```

JPA의 `em.find(Entity, id)` 와 사실상 동일.

## 일반 select와 Identity Map의 상호작용

```python
stmt = select(UserModel).where(UserModel.id == some_id)
a2 = (await session.scalars(stmt)).first()
```

`session.get` 과 달리 일반 select는 **항상 SQL을 발행**한다. 하지만 결과 row를 객체로 hydrate할 때 identity map을 거친다:

1. SQL은 무조건 실행됨 (`SELECT * FROM users WHERE id = ?`)
2. 돌아온 row의 PK로 `(UserModel, some_id)` 키를 만듦
3. Identity Map 조회
4. **있으면** → 기존 인스턴스 반환, row의 컬럼 값들은 **버려짐** (`__init__` 호출 X)
5. **없으면** → 새 인스턴스 hydrate 후 identity map 등록

```python
a1 = await session.get(UserModel, some_id)  # SQL 1회
a2 = (await session.scalars(stmt)).first()  # SQL 1회 더, but a1 is a2

assert a1 is a2  # True — 객체 동일성은 보장
```

### 중요한 함의 — DB가 더 신선해도 메모리 객체가 이긴다

1회차와 2회차 사이에 **다른 트랜잭션이 row를 UPDATE 했더라도**, 2회차에서 받는 `a2` 는 1회차 시점의 값 그대로. SQL은 새 값을 가져왔지만 hydrate 단계에서 버려진다.

이는 unit of work 패턴의 핵심 트레이드오프 — **객체 일관성 vs DB 최신성** 중 일관성을 택한다. JPA에서 `em.find()` 후 JPQL을 날려도 1차 캐시 우선 동작과 동일.

## 강제 갱신 — refresh, expire, populate_existing

DB의 최신 값을 받아야 할 때 쓰는 4가지 도구. 각각 **타이밍**과 **적용 범위**가 다르다.

### refresh — 즉시 강제 reload

```python
a = await session.get(UserModel, some_id)
# ... 다른 트랜잭션이 UPDATE ...
await session.refresh(a)
# SELECT * FROM users WHERE id = ?
# → 결과로 a의 모든 속성을 in-place 갱신
```

핵심:

- **즉시** SELECT 1회 발행 (identity map 우회 — 그게 이 메서드의 존재 이유)
- 객체의 정체성(`id(a)`)은 유지, 속성만 갱신
- JPA의 `em.refresh(entity)` 와 동일

```python
a_before = id(a)
await session.refresh(a)
a_after = id(a)
assert a_before == a_after  # 같은 파이썬 객체, 속성만 갱신
```

#### 왜 identity map은 별도로 갱신할 필요가 없나

같은 PK로 여러 번 조회한 변수는 모두 **같은 파이썬 객체를 가리키는 참조**다:

```python
a = await session.get(UserModel, 1)
b = await session.get(UserModel, 1)  # identity map hit, SQL 없음

assert a is b           # ✅ 같은 객체
assert id(a) == id(b)   # ✅ 같은 메모리 주소
```

Identity map은 본질적으로 `dict[(Mapper, PK), 객체참조]` 구조다 — **"어떤 객체인지"만 기억할 뿐 "그 객체가 어떤 값을 갖는지"는 객체 자신이 들고 있다.** 그래서 `refresh` 로 객체 속성을 in-place 갱신하면, 그 객체를 가리키는 모든 참조에서 자동으로 새 값이 보인다:

```python
# DB에서 다른 트랜잭션이 name="Bob" 으로 UPDATE 후
await session.refresh(a)

print(a.name)   # "Bob"
print(b.name)   # "Bob" — a와 같은 객체니까
print(a is b)   # 여전히 True
```

이 약속("같은 PK = 같은 파이썬 객체") 덕분에 lost update 같은 일관성 문제가 원천적으로 불가능해진다 — JPA 1차 캐시가 보장하는 것과 동일한 가치.

`refresh` / `expire` / 코드에서의 직접 속성 변경 모두 같은 메커니즘으로 동작하고, **identity map entry 자체를 건드리는 건 `expunge` / `close` 처럼 객체를 세션에서 분리할 때뿐**이다.

#### 옵션: attribute_names — 컬럼/관계 좁히기

```python
await session.refresh(a, attribute_names=["status", "updated_at"])
# SELECT status, updated_at FROM users WHERE id = ?
```

큰 BLOB/JSON 컬럼이 있어 전체 reload가 부담될 때, 또는 특정 컬럼만 다시 보고 싶을 때. **관계(relationship) 속성 이름**도 넣을 수 있어서 그 관계만 다시 로드 가능.

#### 옵션: with_for_update — 비관적 락

```python
await session.refresh(a, with_for_update=True)
# SELECT * FROM users WHERE id = ? FOR UPDATE
```

DB 레벨에서 **행 잠금(row lock)** 을 걸어 다른 트랜잭션이 같은 row를 UPDATE/DELETE 하려 하면 대기시킴. JPA의 `LockModeType.PESSIMISTIC_WRITE` 와 동일. 잔액 차감 같은 race condition이 우려되는 작업에 사용.

⚠️ 두 옵션은 **완전히 별개 기능**이다 — `attribute_names` 는 reload할 컬럼 좁히기, `with_for_update` 는 행 잠금. 같이 쓸 수도 있고 따로 쓸 수도 있다.

### expire — 지연 reload (lazy)

```python
session.expire(a)
# 이 시점엔 SQL 없음 — 무효화 마커만 표시

print(a.name)  # ⚠️ 여기서 비로소 SELECT 발생
```

`refresh` 와의 핵심 차이는 **타이밍**:

| 메서드 | SQL 발행 시점 | 동작 |
|--------|-------------|------|
| `await session.refresh(obj)` | **즉시** 1회 | 호출 즉시 SELECT, 객체 in-place 갱신 |
| `session.expire(obj)` | **다음 속성 접근 시** (lazy) | 호출 시점엔 무효화 마커만, 접근 시 SELECT |

언제 어떤 걸:

- "지금 당장 fresh한 값이 필요" → `refresh`
- "나중에 쓸 수도 있고 안 쓸 수도 있음" → `expire` (안 쓰면 SQL 절약)

`session.expire_all()` 은 세션 내 모든 객체에 expire 마커. JPA의 `em.clear()` 와 효과가 비슷 (다만 `em.clear()` 는 detach까지 간다는 차이).

`expire` 의 자세한 메커니즘과 함정은 [[column-loading-and-expire]] 참조.

### expire_on_commit — 세션 단위 자동 정책

per-call 메서드가 아니라 **세션 생성 시 한 번 설정하는 정책**:

```python
async_session = async_sessionmaker(
    engine,
    expire_on_commit=True,   # ← 기본값
)

async with async_session() as session:
    user = await session.get(UserModel, id)
    await session.commit()   # ← 자동으로 user 포함 모든 객체 expire
    print(user.name)         # ⚠️ SELECT 재발행 (expired 상태였으니)
```

| 설정 | 동작 | 의도 |
|------|------|------|
| `True` (기본) | commit 후 세션 안 모든 객체 자동 expire | 보수적 가정 — "트랜잭션 밖 데이터는 stale 할 수 있다" |
| `False` | commit해도 객체 상태 유지 | 명시적 갱신만 — async + API 응답 직렬화 패턴에서 권장 |

특히 async 환경에서는 `False` 가 거의 표준:

```python
async_sessionmaker(engine, expire_on_commit=False)
```

`True` 로 두면 commit 후 응답 직렬화 단계에서 의도치 않은 SELECT 재발행이 일어나고, ORM 객체에 도메인 검증을 넣은 경우 그 검증이 엉뚱한 시점에 폭발한다 ([[orm-domain-separation]] 참조). False로 두고 fresh가 필요한 시점에만 명시적으로 `refresh` / `expire` 를 호출하는 게 안전.

### populate_existing — 쿼리 단위 덮어쓰기

```python
stmt = (
    select(UserModel)
    .where(UserModel.id == some_id)
    .execution_options(populate_existing=True)
)
a = (await session.scalars(stmt)).first()
```

평소 `select` 는 identity map hit 시 hydrate 결과를 버리는데, 이 옵션을 켜면 **그 쿼리 한정으로 identity map에 덮어쓴다**. "이 쿼리는 fresh한 값으로 객체를 갱신할 목적" 임을 명시할 때 사용.

### 도구 선택표

| 상황 | 도구 | JPA 대응 |
|------|------|---------|
| 이 객체 1개를 지금 당장 fresh하게 | `await session.refresh(obj)` | `em.refresh(obj)` |
| 이 객체 1개를 다음 접근 시 fresh하게 | `session.expire(obj)` | (없음) |
| 세션의 모든 객체를 다음 접근 시 fresh하게 | `session.expire_all()` | `em.clear()` (유사) |
| commit할 때마다 자동으로 모두 expire | `expire_on_commit=True` (세션 설정) | (없음) |
| 특정 쿼리 결과로 identity map 덮어쓰기 | `stmt.execution_options(populate_existing=True)` | (없음) |

## 정리

- **Session = 영속성 컨텍스트**, Identity Map = 1차 캐시
- `session.get(Entity, pk)` 는 identity map을 먼저 보고 hit이면 SQL 없음
- 일반 `select` 는 항상 SQL을 실행하지만, **객체 동일성은 identity map이 보장**
- DB 최신성이 필요하면:
  - per-object 즉시 갱신 → `refresh` (옵션: `attribute_names` 로 컬럼 좁히기, `with_for_update` 로 행 잠금)
  - per-object 지연 갱신 → `expire` / `expire_all`
  - 세션 단위 자동 정책 → `expire_on_commit` (async에서는 보통 `False` 권장)
  - 쿼리 단위 덮어쓰기 → `populate_existing`
