---
title: closingIssuesReferences=[] 함정 새 변형 — base=develop PR 의 closing keyword 100% 미발화 (volt #93 확장)
type: knowledge
source_repo: coseo12/astro-simulator
source_issue: 115
captured_at: 2026-05-25
status: refined
tags: [github, auto-close, closing-keyword, base-develop, default-branch, gitflow, closing-issues-references]
related:
  - ../meta/patterns/closing-issues-references-empirical-verify.md
  - ../meta/patterns/sub-agent-finalization-miss-pattern.md
  - ./closing-refs-base-develop-32nd-reproduction.md
  - ./github-auto-close-syntax-h1-h2-hypothesis.md
---

## 요약

`Closes #N` 콜론 없는 정확 문법인데도 `closingIssuesReferences=[]` 발생. 새 변형 발견 — **base=develop (default branch=main 아님) PR 머지 시 GitHub closing keyword 가 결코 발화하지 않음** (gitflow 운영 시). 실증: 본 레포 최근 10 base=develop PR 100% 미발화 (10/10).

## 본문

**가설 검증 통계 (astro-simulator, 2026-05-17)**:

| PR | has_closes_keyword | closing_refs_count |
|----|--------------------|--------------------|
| #497 | true (Closes #473) | **0** |
| #496 | false | 0 |
| #493 | false | 0 |
| #492 | true | **0** |
| #491 | true | **0** |
| #490 | true | **0** |
| #489 | true | **0** |
| #486 | true | **0** |
| #482 | true | **0** |
| #481 | true | **0** |

**키워드 보유 8건 중 8건 (100%) 모두 미발화**. 본 레포는 base=main PR 의 auto-close 실측 안 됨 (release PR 외 적음) — 그러나 GitHub 공식 동작이 default branch 머지 시만 발화하는 것은 명백.

**볼트 #93 (sub-agent auto-close 박제 누락) 과의 관계**:
- volt #93: sub-agent 가 `Closes: #N` 콜론 문법으로 자동 박제 → 미인식 함정
- 본 새 변형: 콜론 없는 정확 문법도 base=develop 이면 미인식

**두 함정 메커니즘 분리**:

| 메커니즘 | 트리거 | 인식 정책 |
|---------|--------|----------|
| **함정 A — 콜론 문법** (volt #93) | `Closes: #N`, `Fixes: #N` | base 무관, 콜론 자체가 인식 차단 |
| **함정 B — base 함정** (본 이슈) | base != default branch | 정확 문법이어도 차단 |

**GitHub 공식 동작** (확인 권장):
- closing keyword 는 PR 머지 시 발화 — **단, default branch (main/master) 머지 시만**
- gitflow 운영 (develop 중간 단계) 환경에서는 모든 closing keyword 미발화
- release PR (`develop → main`) 머지 시점에 누적된 PR 본문의 `Closes #N` 키워드가 main 으로 흘러들어가 발화하는지 — 불명확 (테스트 필요)

**예방 (gitflow 운영 시 의무)**:

1. **수동 close 의무화** — base=develop PR 머지 후 메인 오케스트레이터가 `gh issue close <N>` 실행
2. **메인 마무리 체크리스트 추가** — `auto_close_issue_states` field 에 실제 상태 박제 (volt #24 sub-agent SSoT 의 메인 확장)
3. **PR 본문에 명시 박제** — `Closes #N (수동 close 의무 — base=develop 함정 32회차)`

**가설 추가 검증 (후속 후보)**:
- base=main PR (release/hotfix) 머지 시 closingIssuesReferences 작동 여부 실측
- 작동하면 함정 B 확정 (default branch only)
- 미작동하면 다른 원인 (예: PR template 의 Closes 위치 — 본문 최상단/하단/체크박스 안 등)

**일반화**:
- "auto-close = 기본 동작" 의 GitHub 공식 가정이 gitflow 운영 환경에선 100% 무효
- 모든 base=develop PR 의 auto-close 를 수동 close 로 의무화하는 게 안전
- 메인 오케스트레이터의 마무리 체크리스트 JSON 의 `auto_close_issue_states` field 가 가장 신뢰할 수 있는 verification

## 관련 노트/링크

- 본 세션 사례: PR #497 (#473 D4 CI backstop, 2026-05-17), PR #496 (#480 SSoT drift), PR #486 (#476 Amendment 1) 등
- 함정 누적: CLAUDE.md memory 박제 "closingIssuesReferences=[] 함정 31회차" → 본 변형으로 32회차 시점 메커니즘 분리
- 관련 volt: #93 (sub-agent auto-close 박제 누락, 콜론 문법 함정 메커니즘 A), #24 (sub-agent 검증 ≠ GitHub 박제)
