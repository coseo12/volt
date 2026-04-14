# 기여 가이드

이 볼트는 개발 중 쌓이는 지식과 경험을 공개적으로 축적해 Claude Code agent/skill 개선의 원천 데이터로 쓰기 위한 공간입니다.

## 어떻게 기여하나요

### 1. 이슈로 캡처 (모든 기기에서 가능)

가장 쉬운 방법입니다. 두 가지 경로가 있어요.

**웹 UI** — [새 이슈 만들기](../../issues/new/choose) 에서 `Knowledge` 또는 `Report` 템플릿 선택.

**CLI / AI 에이전트** — `gh` 커맨드로 직접 생성. 세부 규약은 [AGENTS.md](./AGENTS.md) 참고 (라벨·제목 접두사·본문 구조 명세).

### 2. 이슈 분류 가이드

- **Knowledge**: 재사용 가능한 개념, 패턴, 도구 사용법 (시간·맥락 독립적)
- **Report**: 특정 작업에서의 경험, 결정, 회고 (시간·맥락 종속적)
  - 세부 유형: `troubleshooting`, `retrospective`, `research`, `decision`, `feedback`, `pattern`

### 3. PR로 직접 기여

정제된 노트를 바로 제안하고 싶다면 PR도 환영합니다.

- 파일은 `inbox/` 가 아닌 목적지(`notes/` 또는 `meta/*`)에 바로 둠
- frontmatter 표준(아래 참고)을 지킬 것

## Frontmatter 표준

```yaml
---
title:
type: knowledge | report | pattern | decision | feedback
source_repo:       # 출처 레포 (선택)
source_issue:      # 이슈 번호 (이슈 변환 시)
captured_at: YYYY-MM-DD
status: inbox | refined
tags: []
related: []        # 관련 노트 파일명 또는 URL
---
```

## 이슈 → 파일 변환 규칙

이슈가 들어오면 관리자(사람 또는 자동화)가 다음 절차로 처리합니다.

1. `inbox/{slug}.md` 로 파일 생성 (`slug` = 제목의 케밥케이스)
2. frontmatter 채움, `status: inbox`
3. 정제 단계에서 `notes/` 또는 `meta/*` 로 이동, `status: refined`

### 라벨·유형별 목적지 매핑

| 라벨 | `리포트 유형` | 최종 목적지 | frontmatter `type` |
|---|---|---|---|
| `knowledge` | — | `notes/` | `knowledge` |
| `report` | `troubleshooting` / `retrospective` / `research` | `notes/` | `report` |
| `report` | `decision` | `meta/decisions/` | `decision` |
| `report` | `feedback` | `meta/feedback/` | `feedback` |
| `report` | `pattern` | `meta/patterns/` | `pattern` |

## 금지 사항

- 민감정보(비밀키, 토큰, 내부 URL, 고객 데이터) 포함 금지. **이 레포는 public 입니다.**
- 저작권 침해 콘텐츠 게재 금지.

## 질문

이슈나 Discussion 으로 문의 주세요.
