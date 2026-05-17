---
title: silent 가드 약화 의식적 트레이드오프 — 1인 운영 현실 대응 (Amendment 2 사례)
type: knowledge
source_repo: coseo12/astro-simulator
source_issue: 106
captured_at: 2026-05-17
status: inbox
tags: [trade-off, silent-guard, single-operator, threshold-tuning, amendment]
related:
  - ../notes/adr-trigger-discussion-issue-pattern.md
  - ../notes/roi-test-skip-judgment-drift.md
---

## 요약

가드 임계값을 강제하면 1인 개발자에게 매주 [ADR Trigger] 발화 + 결정 강요 → 운영 피로 + silent skip 위험. **임계값을 의식적으로 완화** + **silent 가드 약화 트레이드오프 ADR §결정에 CRITICAL 명시** 하는 패턴. 일반화: 가드 설계는 운영 비용까지 고려한 양면 박제.

## 본문

**사례 (ADR 20260515 Amendment 2, PR #489)**:

**Amendment 1 (PR #486)** — 엄격 임계값:
- Phase 2 = N/M < 33% **AND** N >= 3 → 트리거 발화
- 경과일수 >= 30일 → 트리거 발화

실측: Amendment 1 머지 즉시 Phase 2 = 0/7 = 0% → 첫 [ADR Trigger] #487 박제. **8일째 경과 + 8회차 박제 + 일주일 운영** → 매주 발화 + 결정 강요 → 1인 운영 피로 검출.

**Amendment 2 (PR #489)** — 의식적 완화 (옵션 C):
- N >= 3 → N >= 10
- 30일 → 90일

§결정 박제 (CRITICAL):
> Amendment 2 는 트리거 발화 빈도를 줄이기 위한 의식적 silent skip 완화. **운영 피로 vs silent drift 검출 의 트레이드오프를 1인 운영 현실에 맞게 조정**. silent drift 위험은 best-effort 로 수용 — Phase 2 진행률 health metric 자동 탐지가 보강.

**의의**:
- 가드 임계값은 **운영 비용 + silent drift 위험 + 결정 비용** 3축 최적화
- 1인 운영 / 소규모 팀 / 자동화 부재 도메인에서는 임계값 완화가 의식적 트레이드오프
- 완화 자체를 ADR §결정에 CRITICAL 명시 → 미래 관찰자가 "왜 완화됐는지" 재발견 가능

**대안 (검토 후 미채택)**:
- 옵션 A (현 정책 유지): 운영 피로 누적 → silent skip 자연 발생 위험
- 옵션 B (ADR 폐기): 가드 자체 폐기는 과잉 — 임계값 조정만 필요

**일반화**:
- 가드 도입 PR 의 DoD 에 "운영 비용 추정" 명시 — 예상 발화 빈도 + 결정 시간
- 발화 빈도 ≥ 1/주 → 1인 운영 / 사용자 결정 비용 ≥ 30분/주
- 발화 빈도 ≥ 1/일 → 운영 피로 critical → silent skip 위험

**적용 조건**:
- 가드 임계값 첫 박제 후 1~2주 실측
- 운영 피로 검출 시 의식적 트레이드오프 (Amendment) 박제
- "silent skip 위험" 을 §결정에 박제 — 후속 관찰자가 재발견 시 빠른 재기각

## 관련 노트/링크

- 사례: ADR 20260515 Amendment 2 (PR #489, 2026-05-16)
- 관련 volt: #105 ([ADR Trigger] discussion 패턴), #71 (ROI 5문 체크 한계 — 시각 회귀 사례)
