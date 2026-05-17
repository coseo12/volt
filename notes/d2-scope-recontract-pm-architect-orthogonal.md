---
title: D2 범위 재계약 '설계의 백미' — PM 재계약이 architect 결정과 직교 차원으로 작용한 사례
type: report
source_repo: coseo12/astro-simulator
source_issue: 110
captured_at: 2026-05-17
status: refined
tags: [pm-recontract, scope-renegotiation, architect-pm-orthogonal, dod-revision, design-elegance]
related:
  - ../meta/patterns/pm-sub-agent-multi-turn-round-drift.md
  - ./cross-validate-plan-mode-bypass-guard.md
---

## 배경/상황

복합 가드 (#479 cross-validate plan-mode 우회) 도입 PR 의 architect 단계에서 결정 6 (D/R4/S1/F2/SSoT/Z6) 박제 후, PM 이 D2 범위 재계약 (DoD-2 측정 방법 C 박제) 을 architect 결정과 **직교 차원**에서 진행. PM-architect 워크플로의 직교성이 작동한 retrospective.

## 내용

**시퀀스 (PR #482 #479 cross-validate plan-mode 우회 가드)**:

1. architect → Gemini cross-validate (6 결정점 D/R4/S1/F2/SSoT/Z6) 박제
   - D: snapshot+hash 혼합 비교
   - R4: 자동 롤백
   - S1: outcome 스키마 확장
   - F2: 5 페르소나 SSoT 박제
   - SSoT: 5 페르소나 동일성 검증
   - Z6: ADR Z 패턴 6회차 재사용

2. PM 이 D2 (DoD-2 측정 방법) 범위 재계약 — architect 결정과 **별개 차원**:
   - architect 결정 D2 (snapshot 비교 알고리즘) 와 무관
   - PM 의 D2 = "측정 방법 C (정확 식별자) 박제" — `.claude/skills/create-pr/SKILL.md` 동시 수정

3. dev sub-agent 가 architect D2 (snapshot 비교) + PM D2 (측정 방법 C) 둘 다 구현 — **직교 작용**

**'설계의 백미'**:
- architect D2 와 PM D2 는 같은 라벨 "D2" 지만 **다른 차원의 결정**
- PM 재계약이 architect 결정과 직교 → 충돌 없음, 보완
- 5 페르소나 워크플로의 직교성이 "같은 이름 다른 차원" 분기를 자연스럽게 처리

**워크플로 의의**:
- architect = "어떻게 (how) 구현할 결정"
- PM = "무엇을 (what) DoD 로 박제할 결정"
- 같은 PR 안에서 둘이 직교 → 결정 박제 풍부도 ↑

**일반화 (PM-architect 직교 작용)**:
- PM 재계약과 architect 결정이 같은 라벨 (D1/D2/...) 을 사용해도 직교 차원이면 충돌 없음
- "PM 재계약 = 범위 변경, architect 결정 = 구현 결정" 의 명확 구분
- 5 페르소나 워크플로의 sub-agent 격리가 직교성을 보장

**자기 점검** (drift 위험):
- PM 재계약이 architect 결정 차원을 침범하면 충돌 — 메인 오케스트레이터가 차원 구분 명시 의무
- 같은 라벨 (D1/D2/...) 이 다른 차원을 가리킬 때 PR 본문에 명시 박제 (예: "D2 (PM) = 측정 방법 C, D2 (architect) = snapshot 비교")

## 교훈 / 다음에 적용할 점

- PM-architect 직교 작용 가능 — 5 페르소나 워크플로의 강점
- 같은 라벨 다른 차원은 의식적 박제 (PR 본문에 명시)
- 다음 PR 에서 의도적 직교 활용 시도 — PM 이 architect 결정과 다른 차원에서 의식적 재계약
- harness real-lessons 박제 후보 — "PM-architect 직교 패턴"

## 관련 노트/링크

- 사례: PR #482 (#479 cross-validate plan-mode 우회 가드, 2026-05-15)
- 관련 volt: #34 (PM sub-agent multi-turn 라운드 이탈), #99 (#479 가드)
