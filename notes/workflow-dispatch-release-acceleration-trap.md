---
title: workflow_dispatch 2단계 함정이 의도하지 않은 release 가속 트리거로 작용 (volt #45 후속)
type: report
source_repo: coseo12/astro-simulator
source_issue: 97
captured_at: 2026-05-17
status: refined
tags: [github-actions, workflow-dispatch, release-cadence, gitflow, accidental-trigger]
related:
  - ./workflow-dispatch-default-branch-trap.md
  - ./yaml-on-boolean-coercion-trap.md
  - ./adr-trigger-discussion-issue-pattern.md
---

## 배경/상황

`workflow_dispatch` 트리거는 default branch (main) 에 workflow 파일이 머지된 후에만 discover 된다 (volt [#45](https://github.com/coseo12/volt/issues/45) 박제). 본 사례는 #45 와 같은 메커니즘이 **의도하지 않은 release 가속 트리거**로 작용해 develop 누적 57 커밋을 정상 release v0.16.0 으로 묶어낸 retrospective. 함정의 부정적 효과 (검증 차단) 와 긍정적 효과 (release 리듬 정렬) 가 동시에 관찰된 사례.

## 내용

**시퀀스 (astro-simulator 2026-05-17)**:

1. ADR Z 패턴 health metric 자동 탐지 workflow (`.github/workflows/adr-z-pattern-health-v2.yml`) 박제 — develop 머지 (PR #490 + hotfix-2 PR #491/#492)
2. D2/D3 검증 (수동 `gh workflow run` 발화) 시도 → **HTTP 404 "not found on the default branch"** — workflow_dispatch 함정 #45 재현
3. 검증 차단 → 3 옵션 비교:
   - A. release PR (develop → main) — 57 커밋 함께 main 으로
   - B. cron 자연 발화 대기 (월 09:00 KST) — 약 2주
   - C. workflow 만 main 으로 hotfix cherry-pick — gitflow 위반
4. **옵션 A 선택** → 정상 gitflow release v0.16.0 (60 커밋 누적, 평소 5~10 커밋 PR 단위와 매우 큰 차이)

**부정적 효과** (#45 이미 박제):
- 검증 차단 — D2/D3 (workflow 발화 + 자동 이슈 생성) 가 main 머지까지 지연
- workflow 함정 hotfix 2회 (`'on':` boolean True + 파일명 stale ID)

**긍정적 효과** (본 retrospective 추가):
- **release 자연 시점 정렬** — develop 57 커밋 누적이 "충분히 쌓임" 임계값에 도달했음을 우회로 발견. 정기 release 시점 판단 휴리스틱 부재 상황에서 의도하지 않은 정렬 트리거 작용
- **release 결정 분기 강제 노출** — 검증 차단이 사용자에게 "지금 release 할지" 명시 결정 강요. 자연 진행 시 50~80 커밋까지 누적 후 결정 강요받는 패턴 회피
- **검증 비용 vs release 비용 트레이드오프 노출** — 옵션 B (대기 2주) 가 합리적이었으나 옵션 A (즉시 release) 가 실질 비용이 더 작은 영역 (release 가치 > 대기 가치) 임이 드러남

**메타 패턴**:
- "함정이 트리거가 된다" — 함정의 강제력이 우회보다 정공법을 선택하게 만드는 경우. 일반화: **검증 차단이 정공 결정의 자연 강요로 작동하는 상황**
- 도메인 의존성:
  - feature 누적 ≥ 일정 수준 → 정공법 (release) 비용 < 우회법 (cherry-pick) 비용
  - feature 누적 < 일정 수준 → cron 대기 또는 hotfix 가 합리적

**release 가속의 한계**:
- 모든 검증 차단이 release 가속을 정당화하지 않음 — 본 사례는 **develop 누적 57 커밋 + 정상 release 시점 도달** 조건 우연 일치
- 만약 develop 누적 5~10 커밋이었으면 옵션 B (대기) 또는 C (cherry-pick) 가 합리적

## 교훈 / 다음에 적용할 점

- **검증 차단 직면 시 결정 분기 의무화** — A/B/C 옵션 표 (비용/시점/gitflow 정합성) 명시 후 사용자 명시 결정. "자동으로 정공법" 금지
- **release 누적 모니터링 자동화 후보** — `git rev-list --count origin/main..origin/develop` 가 N 이상이면 dashboard 또는 weekly 리마인더 — release 결정 강제 트리거를 함정에 의존하지 않게 분리 (release-cadence-check workflow 신설 후보)
- harness real-lessons 박제 후보: "함정의 양면성 — 차단 비용 vs 결정 가속 가치"
- volt #45 와 직교: #45 는 함정 자체 + 예방 (default branch 반영 후 실행 검증 DoD 명시), 본 #N 는 함정의 부산물 + retrospective 가치

## 관련 노트/링크

- 본 사례 PR: astro-simulator [#494](https://github.com/coseo12/astro-simulator/pull/494) (release v0.16.0, 2026-05-17)
- 함정 박제: volt [#45](https://github.com/coseo12/volt/issues/45) (workflow_dispatch 2단계 함정)
- hotfix 회로: PR #491 (YAML on quote) / PR #492 (workflow rename `-v2.yml`)
- v0.16.0 release: https://github.com/coseo12/astro-simulator/releases/tag/v0.16.0
