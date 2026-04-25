---
title: developer 후 reviewer/qa 건너뛰기 — headless pixel diff 의 자명 PASS 함정
type: report
source_repo: coseo12/astro-simulator
source_issue: 77
captured_at: 2026-04-25
status: inbox
tags: [agents, workflow, sub-agent, qa, reviewer, headless, pixel-diff, regression-guard]
related:
  - ../meta/patterns/sub-agent-finalization-miss-pattern.md
  - ../notes/swiftshader-headless-pixel-freeze.md
  - ../notes/stale-dev-server-port-collision.md
  - ../meta/patterns/sub-agent-bg-process-leak-pattern.md
  - ../notes/fact-mode-unification-ux-regression.md
---

## 배경/상황

R1 #329 (태양 가시성 복구) 스프린트에서 **developer sub-agent 마무리 직후 메인 오케스트레이터가 reviewer + qa 단계를 건너뛰고 사용자 실 Chrome 검증으로 직행한 워크플로 누락**.

- 사용자가 실 Chrome 검증 중 거대 노란 사각형 (billboard quad ×75) 시각 회귀 발견
- Phase 1 fix (`acfcb74`) 로 차단 + 근본 원인은 별도 이슈 #333 분리
- 그 후 사용자 지적으로 reviewer + qa 단계 정상 복귀
- PR #332 머지 (commit `6e7382e`, 2026-04-25T07:49Z)

관련 자료:

- 이슈 #329 (R1 본 스프린트, close): https://github.com/coseo12/astro-simulator/issues/329
- 이슈 #333 (Phase 2 별도, billboard ×75 분리): https://github.com/coseo12/astro-simulator/issues/333
- PR #332 (R1 머지): https://github.com/coseo12/astro-simulator/pull/332
- reviewer 코멘트: https://github.com/coseo12/astro-simulator/pull/332#issuecomment-4318463711
- qa 코멘트: https://github.com/coseo12/astro-simulator/pull/332#issuecomment-4318484862
- Phase 1 fix 커밋: `acfcb74`
- 신규 회귀 가드: `apps/web/scripts/p329-qa-focus-lod-guard.mjs` (commit `9516b68`)

## 내용

#### 사건 흐름

1. developer sub-agent 가 R1 구현 + headless playwright pixel diff 검증 PASS 보고
2. 메인 오케스트레이터가 reviewer / qa 단계를 거치지 않고 사용자에게 실 Chrome 검증 요청
3. 사용자가 실 Chrome 에서 거대 노란 사각형 (billboard quad ×75) 발견 — 시각 회귀
4. Phase 1 hotfix (`acfcb74`) 로 즉시 차단, 근본 원인은 #333 으로 분리
5. 사용자 지적으로 워크플로 정상화 (reviewer + qa 단계 복귀)

#### 핵심 함정 분석

**developer sub-agent 의 headless playwright pixel diff = baseline 자체이므로 자명 PASS**.

- developer 가 자기 변경분의 baseline 을 자기 변경분으로 비교하면 mismatch 가 0 으로 나옴 — 회귀 신호 0
- 단순 mismatch ≤ 임계 비교만으론 부족
- headless 만으로 PASS 박제하는 흐름은 **CRITICAL #3 (UI 작업 3단계 검증) 위배 + volt #33 (swiftshader headless freeze) 함정의 변형**

QA 단계가 `channel: 'chrome'` 강제 + 사용자 환경 정합성 확보로 회귀를 자동 박제. headless chromium swiftshader 가 동일 회귀 시그니처를 재현함을 실증 — `detect-gpu-tier='c'` 강제 LOD low 경로.

#### 부수 발견 — store-scene 이중 변수 sync

store action (`setSelectedBody`) 와 scene API (`setFocusOrigin`) 가 두 변수 (`selectedBodyId` / `focusBodyIdForAssert`) 를 분리 관리할 때, 한 쪽만 호출되는 경로 (URL parser, programmatic API) 에서 sync 누락 발생 가능. R1 변경에서 노출된 패턴.

## 교훈 / 다음에 적용할 점

#### 1. 메인 오케스트레이터 워크플로 게이트 강제

developer sub-agent 마무리 후 사용자 실 Chrome 검증으로 **직행 금지**. 반드시 다음 순서 준수:

```
developer → reviewer → qa → 사용자 검증 (또는 머지)
```

예외: docs only / chore PR (행동 변화 없음). 행동 변화 판정은 CLAUDE.md "행동 변화 vs 문서 변경 판정 질문" 적용.

#### 2. headless playwright pixel diff 신호 무용성 명시

baseline 자체로 비교하는 회귀 가드는 자명 PASS. 새로운 회귀 시그니처는 별도 자동화 스크립트가 필수:

- sphere/billboard 판별
- `channel: 'chrome'` 강제 (실 Chrome 바이너리)
- 도메인 특화 픽셀 검증 (특정 색상 존재, scene object visibility, 카메라 응답 diff)

단순 mismatch ≤ 임계 비교만으로는 swiftshader freeze + baseline self-compare 함정을 잡지 못함.

#### 3. store-scene 이중 변수 sync 패턴 가드

store action 과 scene API 가 분리된 변수를 관리할 때 다음 중 하나 적용:

- **subscribe + 마운트 직후 1회 sync** 패턴
- **store action 통합** — store 가 scene 콜백 호출을 캡슐화

URL parser / programmatic API 등 우회 경로에서 sync 누락 방지.

#### 4. 적용 영역

- `.claude/agents/` orchestrator 정책: developer sub-agent 반환 후 reviewer/qa 단계 자동 진입 강제
- harness CLAUDE.md "sub-agent 검증 완료 ≠ GitHub 박제 완료" (volt #24) 의 **워크플로 게이트 버전** 으로 보강
- volt #33 (headless swiftshader 함정) 의 **워크플로 누락 변형** 으로 cross-reference

#### 관련 volt 이슈

- #24 (sub-agent 검증 ≠ GitHub 박제)
- #33 (headless swiftshader 함정)
- #46/#52 (background 프로세스 인계)
- #72 (사실 모드 DoD PASS ≠ 제품 동작) — 본 건의 직전 사례
