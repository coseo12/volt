---
title: 단일 세션 multi-release 왕복 = 세션 의도 이탈 시그널
type: knowledge
source_repo: coseo12/astro-simulator / coseo12/harness-setting
source_issue: 63
captured_at: 2026-04-20
status: inbox
tags: [process, session-management, scope-drift, orchestration, meta]
related:
  - ../meta/patterns/sub-agent-finalization-miss-pattern.md
  - ../meta/patterns/pm-sub-agent-multi-turn-round-drift.md
  - ../notes/pnpm-wasm-monorepo-upstream-fixture.md
---

## 요약

단일 세션에서 upstream 레포에 3+ PR 생성 + 릴리스 태그 3회 발생하면 **본래 세션 의도에서 이탈** 했을 가능성이 높다. 이를 감지하는 증상 지표 + 대응 루틴 박제.

## 본문

#### 배경

2026-04-20 세션의 **사용자 의도**: astro-simulator 의 P10-A (Fact Audit 원칙 박제) 착수. **실제 세션 진행**: 시간 80% 가 harness-setting 레포 CI 디버깅으로 소모. harness-setting 3 릴리스 (v2.28.1 hotfix → v2.28.2 pnpm 경로 → v2.29.1 --if-present 수정), 다운스트림 PR #270 에 6 커밋 누적. P10-A 순수 작업은 #269 머지 + #273 교차검증 반영 2건 뿐.

#### 이탈 시그널 (증상 기반)

- **메인 오케스트레이터가 upstream 레포에 PR 3+ 생성** — 이 세션: harness-setting #176, #177, #182, #183 = 4 PR
- **프로젝트 이슈 번호 증가보다 릴리스 태그 증가가 더 많음** — 이 세션: 프로젝트 이슈 4건 (#268, #271, #272, #273) vs 릴리스 2건 (v2.28.2, v2.29.1) + hotfix 1건. 근접.
- **세션 시간 > 본래 작업 예상 시간의 3배** — P10-A 는 1d 예상. 실제 해당 세션 진행 시간은 여러 시간
- **관심사 혼합 커밋** — 같은 세션에 harness update + P10-A 보강 + 교차검증 루틴 + 후속 이슈 분리까지 병렬 진행

#### 권장 대응 루틴

**사전**:
- 인프라 작업 (harness update / tooling migration) 과 **도메인 작업 (Phase 착수) 을 별개 세션으로 분리** 권고. harness update 가 예상 외 실패를 만들면 즉시 별도 세션으로 분할

**중간**:
- 메인 오케스트레이터가 **upstream PR 2개 이상 연쇄 생성** 시 사용자에게 **명시적 선택 요청**: "원 의도 (예: P10-A) 복귀" vs "현 작업 완결"
- 세션 시간이 본래 작업 예상 시간의 **2배 초과** 시 같은 확인 루틴 트리거

**사후**:
- 세션 종료 시 `[회고]` 라벨로 volt 캡처 — "원 의도 충족도 / 부수 작업 범위 / 분할 가능 여부" 3축 평가

#### 본 세션 사후 평가

- **원 의도 P10-A 착수**: 60% 충족 — #269 + #273 머지 완료, 그러나 P10-B 착수는 다음 세션
- **부수 작업 (harness 3 릴리스)**: 완결. 다운스트림 다른 pnpm 프로젝트에도 혜택 (간접 가치)
- **분할 가능 여부**: harness v2.28.1 → v2.28.2 까지는 분할 가능했으나, 발견 연쇄가 즉각 후속 fix 를 요구해 분할 타이밍 놓침

#### 교훈의 일반화

단일 세션에서 "인프라 장애 발견 → upstream 수정 → 다운스트림 재적용" 이 **2회 이상 반복** 되면:

1. 즉시 사용자에게 이탈 고지
2. 원 의도 회복 가능 여부 상담
3. 불가 시 세션 분할 — 현 세션은 "인프라 집중" 으로 재정의, P10-A 는 다음 세션으로

## 관련 노트/링크

- 본 세션 결과물: astro-simulator PR #269/#270/#273, harness-setting v2.28.2/v2.29.1
- 근거: 사용자의 명시 피드백 "정상적인 플로우가 진행이 안되네" (세션 중반)
- 이 패턴은 volt #24 (sub-agent 박제 누락) / volt #34 (sub-agent multi-turn 이탈) 의 **메인 오케스트레이터 버전**
