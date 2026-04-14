# Volt

개발 과정에서 축적되는 지식과 경험을 저장하는 공개 옵시디언 볼트.
궁극적으로는 Claude Code agent/skill 구성을 개선하기 위한 RAG 원천 데이터로 활용.

## 구조

```
inbox/        # 이슈에서 변환된 원본, 미정제
notes/        # 정제된 지식/리포트 (평면 + 태그 기반)
meta/         # agent/skill 개선 재료
  feedback/   #   작업 중 AI 피드백 사례
  patterns/   #   반복 관찰된 워크플로 패턴
  decisions/  #   도구/접근법 선택 근거
templates/    # 옵시디언 템플릿
attachments/  # 이미지 등 첨부 파일
```

## 캡처 워크플로

어디서든 GitHub Issue로 캡처 → 데스크톱 옵시디언에서 정제 → 해당 폴더로 이동.

1. **캡처**: `knowledge` 또는 `report` 이슈 템플릿으로 작성
2. **변환**: 이슈 내용을 `inbox/{slug}.md` 로 저장 (frontmatter에 `source_issue` 기록)
3. **정제**: `notes/` 또는 `meta/*` 로 이동, `status: refined`, 태그·관련 노트 정리
4. **소비**: RAG 파이프라인은 `status: refined` 만 인덱싱

## Frontmatter 표준

```yaml
---
title:
type: knowledge | report | pattern | decision | feedback
source_repo:
source_issue:
captured_at: YYYY-MM-DD
status: inbox | refined
tags: []
related: []
---
```

## 기여

- 사람 기여자: [CONTRIBUTING.md](./CONTRIBUTING.md)
- AI 에이전트(Claude Code 등): [AGENTS.md](./AGENTS.md)
