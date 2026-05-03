---
title: 컨벤션 & 워크플로우
---

# 컨벤션 & 워크플로우

vault 운영을 위한 개인용 가이드. `_meta/` 폴더에 있어 사이트로 빌드되지 않음 (단, git에는 추적되어 GitHub에 백업됨).

## 목차

- [작성 흐름](#작성-흐름)
- [컨벤션](#컨벤션)
- [문서 타입](#문서-타입)
- [공개 게이트](#공개-게이트)
- [ADR 인덱스](#adr-인덱스)
- [사용 도구](#사용-도구)

## 작성 흐름

1. 학습 시작 → `daily/{YYYY-MM-DD}.md`에 키워드 메모
2. 정리할 만큼 이해됨 → 적절한 카테고리에 본문 작성
3. 백링크 활용: `[[관련-문서]]`로 연결
4. Git commit (Conventional Commits)
5. 보안/내용 검토 → frontmatter `publish: true` 추가 (공개 게이트 통과)

### 분류 고민될 때

`inbox/`에 일단 작성 → 나중에 적절한 위치로 이동.

## 컨벤션

- **파일명**: `kebab-case.md`
- **Frontmatter**: 모든 문서 상단에 메타데이터 (`title`, `type`, `tags`, `last_reviewed`, `publish`)
- **목차**: 제목(`# H1`) 직후 `## 목차` 섹션. 마크다운 앵커 링크 사용 (`[제목](#제목)`)
- **백링크**: `[[]]` 사용 (Obsidian 호환)
- **코드 블록**: 언어 명시 (` ```python `)
- **마지막 검토일**: `last_reviewed` 필드로 6개월마다 갱신

## 문서 타입

각 문서는 frontmatter `type:` 필드로 명시한다. 4가지 중 하나로 분류.

| 타입 | 목적 | 예시 |
|------|------|------|
| `concept` | 개념/원리 — "왜 존재하나" | `annotated.md` |
| `pattern` | 사용법/패턴 — "어떻게 쓰나" | `dependency-injection.md` |
| `reference` | 참조/스펙 — 빠른 조회용 | API 옵션 표 |
| `decision` | 의사결정 기록 (ADR) | `0001-*.md` |

> 분류 기준 자체는 작성자의 정신 모델. 사이트 방문자에게 노출되지 않으므로 자유롭게 운용.

## 공개 게이트

기본값은 **비공개**. `publish: true`를 frontmatter에 명시한 노트만 사이트에 노출됨.

```yaml
---
title: ...
publish: true        # 검토 완료 후 명시적으로 추가
---
```

상세 보안 규칙은 프로젝트 루트의 [CLAUDE.md](../../CLAUDE.md) 참조.

### 공개 전 체크리스트

- [ ] 비밀번호/토큰/API 키 없음
- [ ] DB 접속 정보 없음
- [ ] 내부 IP/도메인/호스트명 없음
- [ ] 회사 비공개 정보 없음
- [ ] 개인정보 없음
- [ ] 코드 예시는 placeholder 사용

## ADR 인덱스

- [[../decisions/0001-notes-tooling|0001 — 학습 노트 도구로 Git + Obsidian 선택]]

## 사용 도구

- **Obsidian** — 작성/연결/시각화
- **Quartz** — 정적 사이트 빌드
- **Git** — 버전 관리/백업
- **GitHub Pages** — 사이트 호스팅 (예정)
