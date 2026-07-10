---
title: ADR Z 패턴 [ADR Trigger] 옵션 A 첫 채택 — Phase 2 시작 시그널 retrospective (1 PR + 5 이슈 박제)
type: report
source_repo: coseo12/astro-simulator
source_issue: 116
captured_at: 2026-05-25
status: refined
tags: [adr-trigger, phase-2-start, harness-managed-divergent, option-a, upstream-contribution, retrospective]
related:
  - ./adr-trigger-discussion-issue-pattern.md
  - ./silent-guard-relaxation-trade-off.md
  - ./adr-amendment-form-b-standard.md
  - ./session-dead-wait-guard-3layer.md
---

## 배경/상황

ADR 20260515 (Z 패턴 — harness-managed divergent) §재검토 조건 #5 자동 탐지가 2회 연속 발화. 직전 [#487](https://github.com/coseo12/astro-simulator/issues/487) (Amendment 2, 옵션 C — silent 가드 완화) 채택 1일 만에 [#495](https://github.com/coseo12/astro-simulator/issues/495) 재발화. **silent 가드 무한 사이클 가속 위험 인지 → 옵션 A (Phase 2 일괄 진행) 첫 채택**. Z 패턴 박제 4일 만의 "행동" 첫 박제 사례.

## 내용

**시퀀스 (astro-simulator 2026-05-15 ~ 2026-05-18)**:

| 일자 | 사건 | Phase 2 | 옵션 |
|------|------|---------|------|
| 2026-05-15 | ADR Z 패턴 박제 (#463) | 0/1 = 0% | (트리거 미발화) |
| 2026-05-16 | Amendment 1 (#476, N=3/30일 트리거) → [#487](https://github.com/coseo12/astro-simulator/issues/487) 발화 | 0/7 = 0% | **C** (Amendment 2: N=10/90일 완화) |
| 2026-05-17 | Amendment 3 (#483, health metric 자동 탐지) → [#495](https://github.com/coseo12/astro-simulator/issues/495) 자동 발화 | 0/11 = 0% | (deadline 2026-05-20) |
| 2026-05-18 | 사용자 결정: **옵션 A 채택** | 1/11 ≈ 9.1% | Amendment 4 박제 |

**옵션 A 채택 근거**:

1. **silent 가드 무한 사이클 위험** — 직전 #487 옵션 C 채택 후 1일 만에 재발화. 추가 완화 (N=20/180일 등) 는 ADR 의도 자체 무력화 (Phase 2 = upstream 기여 = 본래 목적).
2. **Z 패턴 본래 의도 정합** — Phase 2 진행 시그널 확보로 "행동" 첫 박제. Phase 2 = 0 유지는 ADR 박제 의도 자체 부정.
3. **단일 세션 실현 비현실 인지** — 옵션 A 의 "6 PR 일괄 작성" 은 1~2주 분량 → **실현 방법 A1** (첫 PR 1건 + 후속 5 이슈 박제) 채택. Phase 2 = 9.1% 시그널 확보 + 점진 진행.

**실현 방법 A1 의 의의**:

- **단일 세션 deadline 충족** (3 영업일 deadline 2026-05-20) — 옵션 C 회피하면서 실제 행동 시그널 박제
- **Phase 2 추적 가능성 확보** — upstream PR 1건 박제 시 `verify-z-pattern-health.mjs` 즉시 9.1% 측정
- **후속 5 이슈로 점진 진행** — 4/11 = 36% 도달까지 추가 3 PR 필요. upstream issues 로 추적

**Amendment 4 박제 내용** (B 형식):

- 사용자 결정 박제 (2026-05-18)
- upstream PR + 후속 5 이슈 cross-link
- 측정 갱신 (0% → 9.1%)
- 회피 전략 (점진 ≥ 33% 도달)
- Amendment 1+2 임계값 (N=10 / 90일) 유지 — silent 완화 거부

**일반화 — [ADR Trigger] 결정 분기 누적 패턴**:

| 트리거 회차 | 사용자 결정 | 의의 |
|------------|------------|------|
| **#487** (1회차) | 옵션 C — 임계값 완화 | silent skip 트레이드오프 의식적 수용 (volt #106) |
| **#495** (2회차) | 옵션 A — 일괄 진행 | "행동" 박제로 silent 가드 무한 사이클 차단 |

직전 옵션 C 채택 후 즉시 재발화하는 패턴이 **옵션 A 전환의 결정 강제 트리거**. 단일 발화는 옵션 C 가 합리적이지만, **연속 발화는 의도 자체 재검토 시그널**.

## 교훈 / 다음에 적용할 점

- **[ADR Trigger] 결정 분기 의무 시 직전 결정 회수 고려** — 옵션 C (완화) 채택 후 같은 트리거 재발화 시 옵션 A (행동) 전환 우선 검토
- **Phase 2 진행률 9.1% 도 시그널 가치** — 임계값 33% 미달이어도 0% → 9% 변화 자체가 ADR 의도 정합 박제
- **단일 세션 실현 방법 A1 패턴** — 큰 작업 (6 PR 분량) 을 "첫 PR + 후속 5 이슈" 로 분할하여 deadline 충족 + 점진 진행 양립
- **연속 발화는 결정 강제 트리거** — silent 완화 사이클 차단의 자연 신호
- harness real-lessons 박제 후보 — ADR §재검토 조건 결정 분기 운영 패턴

## 관련 노트/링크

- 본 retrospective 사례: astro-simulator [#495](https://github.com/coseo12/astro-simulator/issues/495) (2026-05-18 옵션 A 채택)
- 직전 사례: [#487](https://github.com/coseo12/astro-simulator/issues/487) (2026-05-16 옵션 C 채택)
- ADR Amendment 4 PR: [astro-simulator#498](https://github.com/coseo12/astro-simulator/pull/498)
- Phase 2 첫 upstream PR: [harness-setting#248](https://github.com/coseo12/harness-setting/pull/248)
- Phase 2 후속 5 이슈: harness-setting #249~#253
- 관련 volt: [#105](https://github.com/coseo12/volt/issues/105) ([ADR Trigger] discussion 패턴), [#106](https://github.com/coseo12/volt/issues/106) (silent 가드 약화 트레이드오프), [#104](https://github.com/coseo12/volt/issues/104) (ADR Amendment B 형식)
