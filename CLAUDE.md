# garden-dev-notes

개인 개발 지식 베이스. Obsidian vault + Quartz 정적 사이트 (GitHub Pages 공개).

## 📚 참고 문서

노트 작성·수정을 도울 때 다음 두 문서를 **반드시 함께 참조**:

| 문서 | 내용 | 우선순위 |
|------|------|----------|
| **이 파일 (`CLAUDE.md`)** | 보안 규칙 (시크릿/내부정보 차단), 공개 게이트(`publish: true` 옵트인), 빌드 제외 폴더 | 🔒 필수 — 위반 절대 금지 |
| **`content/_meta/conventions.md`** | 작성 흐름(daily → category → publish), 파일명·frontmatter·목차·백링크 등 스타일 컨벤션, 문서 타입, 사용 도구, ADR 인덱스 | 📐 권장 — 일관성 유지 |

새 노트를 만들거나 기존 노트를 손볼 때는 두 문서의 규칙을 먼저 확인 후 진행한다.

## 🔒 보안 규칙 (절대 위반 금지)

**이 repo는 public이고, `content/` 안의 노트는 GitHub Pages를 통해 인터넷 전체에 공개됩니다.**

다음 정보는 어떤 노트에도 **절대 포함하지 마세요**:

- 비밀번호, API 키, 토큰, 시크릿 (AWS keys, GitHub tokens, OAuth secrets 등)
- DB 접속 정보 (host, port, user, password, connection string, JDBC URL)
- 내부 인프라 정보 (사설 IP, 내부 도메인, 호스트명, VPN 주소)
- 회사 비공개 정보 (미공개 코드, 내부 정책, 제품 로드맵, 매출 등)
- 개인정보 (이메일 주소, 전화번호, 주민번호, 실명 매핑 등)

### 예시 코드 작성 시 항상 placeholder 사용

❌ 잘못된 예시:
```bash
mysql -h 10.0.1.5 -u admin -p'P@ssw0rd!'
psql "postgresql://user:pass@prod-db.internal:5432/main"
export AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
```

✅ 올바른 예시:
```bash
mysql -h <DB_HOST> -u <DB_USER> -p'<DB_PASSWORD>'
psql "postgresql://<USER>:<PASSWORD>@<HOST>:<PORT>/<DB_NAME>"
export AWS_ACCESS_KEY_ID=<YOUR_AWS_ACCESS_KEY_ID>
```

### 실수로 시크릿을 커밋한 경우

단순히 다음 커밋에서 삭제하는 걸로는 부족합니다 — git history에 영구히 남으므로:

1. **즉시 해당 시크릿/비밀번호/토큰을 로테이션** (가장 중요)
2. `git filter-branch` 또는 `git-filter-repo`로 history에서 제거
3. force push로 원격 갱신
4. 다른 협업자가 있다면 통보

## 빌드 제외 폴더

`quartz.config.ts`의 `ignorePatterns`에서 관리:

- `_meta/` — vault 운영 메타데이터 (conventions, 가이드 등) — git 추적 ✅
- `private/` — 진짜 로컬 전용 (`.gitignore`에도 포함 — git 추적 ❌)
- `templates/` — 노트 템플릿 (사이트에 노출 불필요)
- `daily/` — 일일 노트 (정제 안 된 캡처)
- `inbox/` — 미정리 캡처 노트
- `.obsidian/` — Obsidian 워크스페이스 설정

## 🚦 공개 게이트 — `publish: true` 옵트인

**기본값은 비공개**입니다. ignorePatterns로 제외되지 않은 폴더(architecture, concepts, decisions 등)에 있더라도, frontmatter에 `publish: true`가 명시된 노트만 사이트에 노출됩니다.

`quartz.config.ts`의 `ExplicitlyPublished` 커스텀 필터가 검사합니다.

### 새 노트 작성 시 워크플로우

1. 노트 작성 — `publish` 필드 없거나 `false` (기본 비공개)
2. 본인이 보안/내용 검토 완료
3. frontmatter에 `publish: true` 추가 → 사이트에 노출

### 공개용 frontmatter 예시

```yaml
---
title: 노트 제목
type: concept
tags: [...]
last_reviewed: 2026-05-03
publish: true        # ← 검토 완료 후 명시적으로 추가
---
```

### 비공개 유지하려면

`publish` 필드를 **생략하거나** `false`로 설정 (둘 다 동일하게 비공개).

```yaml
---
title: 작성 중인 노트
publish: false       # 생략해도 같음
---
```

### 검토 체크리스트 (publish: true 추가 전)

- [ ] 비밀번호/토큰/API 키 없음
- [ ] DB 접속 정보 없음
- [ ] 내부 IP/도메인/호스트명 없음
- [ ] 회사 비공개 정보 없음
- [ ] 개인정보 없음
- [ ] 코드 예시는 placeholder 사용

## 구조

```
garden-dev-notes/
├── content/              # Obsidian vault (사이트 빌드 대상)
│   ├── architecture/
│   ├── concepts/
│   ├── decisions/
│   └── ...
├── quartz/               # Quartz 빌드 시스템 (수정 X)
├── quartz.config.ts      # 사이트 설정 (제목, baseUrl, ignorePatterns 등)
├── quartz.layout.ts      # 레이아웃 컴포넌트 구성
└── CLAUDE.md             # 본 파일
```

## 일상 워크플로우

```bash
# 로컬 미리보기
npx quartz build --serve

# GitHub Pages 배포 (push만 하면 Actions가 자동 빌드)
git add .
git commit -m "notes: ..."
git push
```
