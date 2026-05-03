---
title: Architecture Decision Records
publish: false
---

# Architecture Decision Records (ADR)

기술적 의사결정의 **컨텍스트와 근거**를 기록한다.

## 왜 작성하나

- 6개월 후 "왜 이렇게 했지?"에 답하기 위해
- 같은 논쟁의 반복을 막기 위해
- 학습 진화 과정을 보존하기 위해

## 작성 규칙

- 파일명: `NNNN-짧은-제목.md` (4자리 ID, 0001부터)
- frontmatter `status`: `proposed` / `accepted` / `deprecated` / `superseded`
- 변경 시 새 ADR 작성 + 이전 ADR을 `superseded`로 표시

## 표준 섹션

```markdown
## 컨텍스트          왜 결정이 필요했나
## 고려한 대안       무엇을 비교했나
## 결정              무엇을 골랐나
## 근거              왜 이걸 골랐나
## 결과 / 트레이드오프  얻은 것과 포기한 것
## 후속 영향          다른 결정에 미치는 영향
```

## 인덱스

- [0001 — 학습 노트 도구로 Git + Obsidian 선택](0001-notes-tooling.md)
