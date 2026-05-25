---
title: 학습 노트 감사 보고서 — 2026-05-25
type: audit
last_reviewed: 2026-05-25
publish: false
---

# 학습 노트 감사 보고서 — 2026-05-25

## 컨텍스트

`content/_meta/note-writing-patterns.md`의 12가지 패턴 + `content/_meta/conventions.md`의 표면 스타일 가이드를 기준으로, `architecture/`, `concepts/`, `language/`, `framework/`, `decisions/`, `database/`, `infrastructure/`, `tools/` 디렉토리 안의 학습 노트 17개를 감사한 결과.

수정은 일절 수행되지 않았다. 이 문서는 **다음 세션에서 그대로 들고 작업할 수 있도록** 액션 아이템을 정리한 것.

## 사용 방법

다음 세션 시작 시 이 파일 경로(`content/_meta/audits/2026-05-25-learning-notes-audit.md`)를 알려주고 다음 중 하나의 지시 형태로 작업 범위 지정:

- "**Quick wins 3건만** 처리해" → 아래 [Quick wins](#quick-wins-5분-이내) 섹션
- "**High priority** 두 건 처리해" → [High priority](#high-priority) 섹션
- "**파일 X.md 항목**만 처리해" → 해당 파일의 액션 아이템
- "**우선순위 무관, 위에서부터** 처리해" → 이 문서 순서대로

각 액션 아이템에는 다음이 포함된다:
- 변경 대상 파일과 라인 (가능한 한 구체)
- 변경 전/후 (가능한 경우)
- 변경 후 검증 방법

수정 후에는 이 문서의 해당 체크박스를 체크하고, 모든 항목 처리 시 파일을 `content/_meta/audits/archive/`로 이동하거나 frontmatter에 `status: done`을 추가하는 식으로 종결.

---

## Quick wins (5분 이내)

각각 단발성, 의존성 없음, 즉시 효과.

### [x] QW-1 — `record.md`의 깨진 `sealed-interface` 백링크 제거

**파일**: `content/language/java/record.md`  
**라인**: 6

**변경 전**:
```yaml
related:
  - "[[sealed-interface]]"
  - "[[object-comparison]]"
  - "[[../../architecture/ddd/value-object]]"
  - "[[../../architecture/ddd/domain-behavior-in-vo]]"
  - "[[../../architecture/ddd/immutability-benefits]]"
```

**변경 후**: `sealed-interface` 줄 삭제 (해당 노트가 작성되면 그때 추가).

**검증**: `grep -rn "sealed-interface" content/` 가 0건이어야 함.

---

### [x] QW-2 — `validators.md`의 깨진 `v1-vs-v2` 백링크 처리 (옵션 A로 처리)

**파일**: `content/framework/pydantic/validators.md`  
**라인**: 6, 243

**변경 전 (라인 6)**:
```yaml
related:
  - "[[v1-vs-v2]]"
  - "[[language/python/typing/annotated]]"
```

**변경 전 (라인 243)**:
```markdown
자세한 v1/v2 차이는 [[v1-vs-v2]] 참조.
```

**선택지**:
- (A) 두 군데 모두 제거 — `related`에서 `v1-vs-v2` 줄 삭제, 본문 라인 243은 문장 자체를 삭제하거나 "위 표 [v1과의 주요 차이 정리] 참조"로 변경
- (B) stub 작성 — `content/framework/pydantic/v1-vs-v2.md` 빈 스켈레톤 만들고 `publish: false` (나중에 채우기)

**권장**: (A). 본문 위 `## v1과의 주요 차이 정리` 표가 이미 v1↔v2 차이를 충분히 다루므로 별도 노트 불필요.

**검증**: `grep -rn "v1-vs-v2" content/` 가 0건이어야 함.

---

### [x] QW-3 — `immutability-benefits.md` 제목에서 "— Address 예제" 삭제

**파일**: `content/architecture/ddd/immutability-benefits.md`  
**라인**: 2, 13

`note-writing-patterns.md`가 직접 예시로 든 안티패턴(`"Money 예제"` 부제)과 같은 패턴.

**변경 전**:
```yaml
title: VO 불변성이 가져오는 안전성 — Address 예제
```
```markdown
# VO 불변성이 가져오는 안전성 — Address 예제
```

**변경 후**:
```yaml
title: VO 불변성이 가져오는 안전성
```
```markdown
# VO 불변성이 가져오는 안전성
```

**부가 작업**: `value-object.md`의 본문에서 이 파일을 참조하는 표시 텍스트도 같이 정리.
```bash
grep -n "Address 예제" content/architecture/ddd/value-object.md
```
라인 189, 363에 `[[immutability-benefits|VO 불변성이 가져오는 안전성 — Address 예제]]` 형태가 있음 → `|` 뒤 alias도 `VO 불변성이 가져오는 안전성`으로 정리.

**검증**: `grep -rn "Address 예제" content/` 결과가 0건이거나 코드 본문 내 설명용으로만 남아있어야 함.

---

## High priority

### [ ] H-1 — `SQLAlchemy Session과 Identity Map.md` 파일명 정상화

**현재**: `content/framework/sqlalchemy/SQLAlchemy Session과 Identity Map.md`  
**문제**: 공백 + 한영 혼용 + 폴더명(`sqlalchemy/`)과 prefix 중복. `conventions.md`의 `kebab-case.md` 규칙 정면 위반. URL 인코딩(`%20`)이 GitHub Pages에서 추악함.

**제안 파일명**: `session-and-identity-map.md`  
(폴더가 `sqlalchemy/`이므로 SQLAlchemy prefix 불필요)

**작업 순서**:

1. **rename** (`git mv`로 히스토리 보존):
   ```bash
   git mv "content/framework/sqlalchemy/SQLAlchemy Session과 Identity Map.md" \
          "content/framework/sqlalchemy/session-and-identity-map.md"
   ```

2. **백링크 갱신** — 다음 1군데:
   - `content/framework/sqlalchemy/query-execution-and-fetching.md:180`  
     변경 전: `session 동작과 identity map은 [[SQLAlchemy Session과 Identity Map]] 참조.`  
     변경 후: `session 동작과 identity map은 [[session-and-identity-map|SQLAlchemy Session과 Identity Map]] 참조.`  
     (display alias로 표시 텍스트는 유지)

3. **본문 H1 제목은 유지 가능** (파일명만 바뀜) — `# SQLAlchemy Session과 Identity Map`은 그대로 둬도 됨

**검증**:
```bash
grep -rn "SQLAlchemy Session과 Identity Map" content/
```
→ 새 파일의 frontmatter `title:`과 본문 `# H1`, 그리고 query-execution-and-fetching의 alias만 남아야 함. wikilink `[[...]]` 형태로는 0건.

---

### [ ] H-2 — `clean-architecture/overview.md`의 "명시적 vs 암묵적 결합" 분리

**파일**: `content/architecture/clean-architecture/overview.md`  
**현재 라인 범위**: 약 232~376 (섹션 `## 명시적 의존 vs 암묵적 결합`)  
**문제**: 약 150줄의 자족 시나리오 (Domain enum 공유 패턴 → 두 차원 비교 → 두 해석 → 판단 기준). `value-object.md` → `domain-behavior-in-vo.md` / `immutability-benefits.md` 분리와 동일한 응집 신호.

**작업 계획**:

1. **새 파일 생성**: `content/architecture/clean-architecture/explicit-vs-implicit-coupling.md`
   - frontmatter:
     ```yaml
     ---
     title: 명시적 의존 vs 암묵적 결합
     type: concept
     tags: [architecture, clean-architecture, coupling, dependency-rule]
     related:
       - "[[overview]]"
     last_reviewed: 2026-05-25
     publish: false
     ---
     ```
   - 부모 노트(`overview.md`)의 `## 명시적 의존 vs 암묵적 결합` 섹션 전체를 이 파일로 옮기되, 자기 자신의 도입부와 `## 관련 노트` 섹션 추가
   - 헤딩 한 단계 승격 (overview의 H3 → 새 파일의 H2)

2. **부모 노트(`overview.md`) 변경**:
   - `## 명시적 의존 vs 암묵적 결합` 섹션을 짧은 개요 + 자식 노트 참조로 축약
   - 예시 1~2줄 + "...자세한 사례와 두 해석 비교는 [[explicit-vs-implicit-coupling]] 참조"
   - `related: []` → `["[[explicit-vs-implicit-coupling]]"]` 추가

3. **양방향 백링크 검증**: 새 파일과 overview 양쪽 frontmatter `related`에 서로 등록.

**검증**:
- `wc -l content/architecture/clean-architecture/overview.md` 가 분리 전 515 → 분리 후 ~370줄 정도여야 함
- Obsidian 그래프 뷰에서 두 노드가 서로 연결되는지

**예상 작업량**: 45분+ (분리 + 본문 헤딩 재구조화 + 백링크 그물)

---

## Medium priority

### [ ] M-1 — `protocol.md`의 `[[architecture/clean-architecture]]` 링크 수정

**파일**: `content/language/python/typing/protocol.md`  
**라인**: 7

실제 파일은 `clean-architecture/overview.md`이므로 `[[architecture/clean-architecture]]`는 Obsidian/Quartz 모두에서 깨진 링크.

**변경 전**:
```yaml
related:
  - "[[annotated]]"
  - "[[architecture/clean-architecture]]"
```

**변경 후**:
```yaml
related:
  - "[[annotated]]"
  - "[[../../../architecture/clean-architecture/overview]]"
```

(또는 H-2 분리 작업 후 더 적절한 자식 노트로 변경)

**부가 작업**: 반대쪽 양방향화 — `clean-architecture/overview.md`의 `related: []`에 `protocol`을 추가.

**검증**: Obsidian 그래프 뷰에서 protocol ↔ clean-architecture 양방향 엣지 확인.

---

### [ ] M-2 — `dependency-injection.md` ↔ `protocol.md` 양방향 백링크

**현재 상태**: `dependency-injection.md` → `protocol.md`는 frontmatter `related`에 있음 (단방향).  
**해야 할 작업**: `protocol.md`의 `related`에 `dependency-injection`도 추가.

**파일**: `content/language/python/typing/protocol.md`  
**라인**: 5~7 (`related:` 블록)

**변경 후 예시**:
```yaml
related:
  - "[[annotated]]"
  - "[[../../../architecture/clean-architecture/overview]]"
  - "[[../../../framework/fastapi/dependency-injection]]"
```

**검증**: `grep -A 5 '^related:' content/language/python/typing/protocol.md` 에 dependency-injection 포함.

---

### [ ] M-3 — sqlalchemy 4파일의 `related` frontmatter 채우기

본문에는 양방향 wikilink가 다 있는데 frontmatter `related`가 비어있어서 그래프 뷰 우선순위 신호가 약함.

#### M-3a — `session-and-identity-map.md` (H-1 처리 후)
- 추가할 related: `column-loading-and-expire`, `orm-domain-separation`, `query-execution-and-fetching`

#### M-3b — `column-loading-and-expire.md`
- 추가할 related: `orm-domain-separation`, `session-and-identity-map` (또는 rename 전 이름)

#### M-3c — `orm-domain-separation.md`
- 추가할 related: `column-loading-and-expire`, `session-and-identity-map`

#### M-3d — `query-execution-and-fetching.md`
- 추가할 related: `session-and-identity-map`

**검증**: 4개 파일 모두 frontmatter `related:` 블록에 최소 2개 이상의 sqlalchemy 노트 등록.

---

### [ ] M-4 — H2 헤딩의 em-dash 부제 일괄 정리

`note-writing-patterns.md` 패턴 5: "헤딩에 설명 부제를 매달지 마라".

영향받는 파일과 헤딩(주요만 발췌):

#### `content/framework/pydantic/validators.md`
- `## 모드 (mode) — 실행 시점 제어` → `## 모드`
- `## ValidationInfo — 외부 정보 접근` → `## ValidationInfo`
- `## context — 외부 정보 주입` → `## context`
- `## 컨테이너 요소 검증 — `Annotated` 합성` → `## 컨테이너 요소 검증`

#### `content/framework/sqlalchemy/column-loading-and-expire.md`
- `## load_only — 일부 컬럼만 가져오기` → `## load_only`
- `## 미로드 컬럼 접근 — 자동 lazy load와 N+1` → `## 미로드 컬럼 접근`
- `## expire — 객체를 껍데기로 되돌리기` → `## expire`

#### `content/framework/sqlalchemy/query-execution-and-fetching.md`
- `## select은 빌더일 뿐 — deferred execution` → `## select은 빌더일 뿐`
- `## .first 함정 — 자동 LIMIT을 안 붙인다` → `## .first 함정`

#### `content/framework/fastapi/dependency-injection.md`
- `## 의존성 별칭 모음 (`deps.py` 패턴)` → `## 의존성 별칭 모음`
- `## yield 기반 의존성 (리소스 관리)` → `## yield 기반 의존성`
- `## 의존성 캐싱 (같은 요청 내)` → `## 의존성 캐싱`
- `## 싱글톤이 필요할 땐 — `lifespan`` → `## 싱글톤이 필요할 땐`
- `## 테스트 — `dependency_overrides`` → `## 테스트`

#### `content/language/java/object-comparison.md`
- `## == — 참조 동등성` → `## ==` (또는 `## 참조 동등성 (==)` 고려)
- `## equals — 값 동등성` → `## equals`
- `## hashCode — equals의 짝` → `## hashCode`
- `## compareTo — 자연 순서` → `## compareTo`
- `## Comparator — 외부 비교 규칙` → `## Comparator`

> **주의**: H2를 바꾸면 같은 파일의 `## 목차` 마크다운 앵커 링크도 깨진다. 헤딩과 목차를 같이 수정할 것. 패턴이 단순하니 `sed`보다 Edit 도구로 1쌍씩 처리 권장.

**검증** (예시):
```bash
# em-dash 헤딩이 줄었는지 확인
grep -c "^## .* — " content/framework/pydantic/validators.md
# 목차 앵커가 헤딩과 일치하는지 (수동 확인)
```

**점진 전략 가능**: 한 번에 다 처리하지 않고 파일별로 next-touch 시 같이 처리해도 됨.

---

### [ ] M-5 — 제목(H1) em-dash 부제 정리

패턴 3 위반. 영향받는 파일:

| 파일 | 현재 제목 | 권장 제목 |
|------|----------|-----------|
| `value-object.md` | `Value Object — 값으로 다루는 객체` | `Value Object` |
| `record.md` | `Java Record — 불변 값 객체를 한 줄로` | `Java Record` |
| `object-comparison.md` | `Java 객체 비교 — ==, equals, hashCode, compareTo` | `Java 객체 비교` |
| `protocol.md` | `Protocol — 구조적 서브타이핑` | `Protocol` |
| `column-loading-and-expire.md` | `부분 로드와 객체 만료 — load_only, expire, lazy column` | `부분 로드와 객체 만료` |
| `query-execution-and-fetching.md` | `쿼리 실행과 결과 처리 — select, execute, scalars` | `쿼리 실행과 결과 처리` |

각 파일에서 frontmatter `title:` 1줄과 본문 `# H1` 1줄, 두 군데 수정.

**판단 보류 옵션**: `_meta/note-writing-patterns.md` 자체도 `학습 노트 작성 패턴 — 반복 편집으로 정제하기` 부제를 가지고 있음. 만약 컨벤션 작성자(본인)가 부제를 의도적으로 허용하는 입장이면 이 항목은 **패턴 5만 적용, 패턴 3은 보류**도 일관된 선택지.

→ 결정 필요: "title em-dash 부제 허용 / 불허 / 케이스별" 중 어느 정책으로 갈지. 결정 결과를 `note-writing-patterns.md`의 패턴 3에 명시화 권장.

---

## Low priority

### [ ] L-1 — `type: example` 분류 결정

**해당 파일**: `domain-behavior-in-vo.md`, `immutability-benefits.md`  
**현재**: `type: example` 사용  
**문제**: `conventions.md`는 `concept` / `pattern` / `reference` / `decision` 4종만 정의

**선택지**:
- (A) `conventions.md`에 `example` 추가 — "부모 개념의 풀 시나리오 예제 노트" 정의 추가
- (B) 두 파일을 `pattern`으로 변경
- (C) 두 파일을 `concept`으로 변경

**권장**: (A). 두 노트의 성격(부모 개념의 풀 시나리오)이 4종 중 어느 것에도 정확히 안 맞고, 같은 패턴이 앞으로도 반복될 가능성 큼.

작업:
- `conventions.md`의 `## 문서 타입` 표에 `example` 행 추가
  ```
  | `example` | 부모 개념의 풀 시나리오 예제 | `domain-behavior-in-vo.md` |
  ```

---

### [ ] L-2 — `0001-notes-tooling.md` frontmatter 정합성

**파일**: `content/decisions/0001-notes-tooling.md`  
**현재**: ADR 표준 따라 `tags`, `last_reviewed` 없이 `date`, `status`만 사용  
**문제**: 다른 학습 노트와 일관성 안 맞음

**선택지**:
- (A) ADR은 예외임을 `decisions/README.md`에 명시 → 0001 파일은 그대로 둠
- (B) ADR도 `tags`, `last_reviewed` 추가 (기존 `date`, `status`와 공존)

**권장**: (A). ADR은 별도 문서 종류라 frontmatter 스킴이 달라도 자연스러움.

작업:
- `decisions/README.md`의 `## 작성 규칙` 섹션에 한 줄 추가:
  > ADR frontmatter는 학습 노트와 달리 `id`, `date`, `status`를 사용한다 (`tags`, `last_reviewed`는 ADR에선 의미가 적어 생략).

---

### [ ] L-3 — H1 첫 문단을 conventions.md의 권장 형태에 맞추기

`note-writing-patterns.md`는 명시적으로 강제하지 않지만, value-object.md 같은 모범 노트들은 H1 직후 `## 목차`가 바로 오는 패턴.

#### 예외 후보
- `SQLAlchemy Session과 Identity Map.md` (현 파일명 기준): H1 직후 짧은 도입 단락이 먼저 옴, 그 뒤 `## 목차`
- `column-loading-and-expire.md`: 같음
- `orm-domain-separation.md`: 같음
- `query-execution-and-fetching.md`: 같음

이 4개 sqlalchemy 노트는 "한 줄 정의" 패턴 대신 짧은 도입 문단 사용. 일관성 차원에선 `한 줄 정의` 헤딩 추가 검토.

**판단 보류**: 강제는 아님. 작성자 취향. 다음 편집 때 결정.

---

### [ ] L-4 — vault 전체 백링크 그래프 시각 점검

모든 수정이 끝난 후 Obsidian에서:
1. 그래프 뷰 열기 (전체)
2. 고립 노드 (orphan) 확인 → `same-origin-vs-same-site.md`처럼 정말 단독인지, 아니면 연결을 빠뜨렸는지 검토
3. 양방향 엣지 vs 단방향 화살표 비율 확인

도구는 Obsidian 그래프 뷰만 있으면 충분. 5분.

---

## 메타 컨벤션 갱신 후보 (별도 의사결정 필요)

이 감사가 드러낸, 컨벤션 자체에 대한 결정 사항:

1. **`type: example` 5번째 타입을 공식화하나?** → L-1 참조
2. **H1 em-dash 부제 정책 명시화** → M-5 참조 (note-writing-patterns.md 패턴 3 본문에 케이스 분류 추가 권장)
3. **ADR frontmatter 스킴 예외 명문화** → L-2 참조 (decisions/README.md에 추가)
4. **sqlalchemy 폴더처럼 폴더가 한 주제일 때 파일 prefix 정책** → "폴더명이 주제를 정의하면 파일 prefix는 폴더명을 반복하지 않는다" 등으로 conventions.md 파일명 규칙에 추가 가능

---

## 부록 — 감사 시점 파일 인벤토리

```
content/architecture/clean-architecture/overview.md
content/architecture/ddd/domain-behavior-in-vo.md           ← publish: true
content/architecture/ddd/immutability-benefits.md
content/architecture/ddd/value-object.md
content/concepts/web/same-origin-vs-same-site.md
content/decisions/0001-notes-tooling.md
content/decisions/README.md                                 ← 메타, 평가 대상 외
content/framework/fastapi/dependency-injection.md
content/framework/pydantic/validators.md
content/framework/sqlalchemy/SQLAlchemy Session과 Identity Map.md  ← publish: true, 파일명 위반
content/framework/sqlalchemy/column-loading-and-expire.md
content/framework/sqlalchemy/orm-domain-separation.md
content/framework/sqlalchemy/query-execution-and-fetching.md
content/language/java/object-comparison.md
content/language/java/record.md                             ← publish: true
content/language/python/typing/annotated.md
content/language/python/typing/protocol.md
```

`database/`, `infrastructure/`, `tools/` 디렉토리는 감사 시점에 `.md` 파일 0건.

---

## 처리 요약 (작업 후 갱신용)

| 우선순위 | 항목 수 | 처리 완료 |
|---------|---------|-----------|
| Quick wins | 3 | 3 |
| High | 2 | 0 |
| Medium | 5 | 0 |
| Low | 4 | 0 |
| 메타 컨벤션 결정 | 4 | 0 |
| **합계** | **18** | **3** |

작업이 끝나면 위 표의 "처리 완료" 컬럼을 갱신하고, 모든 항목 종결 시 이 파일을 `content/_meta/audits/archive/2026-05-25-learning-notes-audit.md`로 이동하거나 frontmatter에 `status: done` 추가.
