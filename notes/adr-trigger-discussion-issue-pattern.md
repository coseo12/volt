---
title: ADR §재검토 조건 트리거 → [ADR Trigger] discussion 이슈 자동 박제 + 3 영업일 deadline 의무
type: knowledge
source_repo: coseo12/astro-simulator
source_issue: 105
captured_at: 2026-05-17
status: refined
tags: [adr, trigger, discussion-issue, deadline, automation, workflow]
related:
  - ./workflow-dispatch-default-branch-trap.md
  - ./yaml-on-boolean-coercion-trap.md
  - ./workflow-dispatch-release-acceleration-trap.md
  - ./silent-guard-relaxation-trade-off.md
  - ./adr-z-pattern-option-a-first-adoption.md
---

## 요약

ADR §재검토 조건이 발화하면 (자동 탐지 workflow 또는 수동 인지) **[ADR Trigger] 이슈를 즉시 박제** + **3 영업일 deadline** + **A/B/C 결정 분기 명시** 패턴. 사용자 결정 분기 위임의 표준화 + ADR mutation 책임 명시화.

## 본문

**패턴 (ADR Z 패턴 자동 탐지 workflow 박제 사례)**:

1. ADR 본문에 §재검토 조건 N 명시 (예: "Phase 2 = N/M < 33% OR 경과일수 ≥ 30")
2. 자동 탐지 workflow 또는 수동 인지 시 발화
3. **[ADR Trigger] 이슈 자동/수동 생성** — title prefix `[ADR Trigger]`, label `type:adr-trigger` (또는 `documentation` + `priority:high`)
4. 이슈 본문 표준 구조:
   - §배경 — workflow path + 발화 시각 + 임계값 충족 조건
   - §임계값 발화 결과 — 측정 수치 + 임계값 + 충족 여부
   - §의무 — 3 영업일 내 결정 분기 (A/B/C)
   - §Cross-link — ADR path + workflow path + 직전 [ADR Trigger]

**3 영업일 deadline 의무**:
- ADR mutation 책임의 시간 박제 — 결정 지연 자체가 의사결정
- 영업일 (workday) — 주말 제외 (월~금 카운트)
- deadline 경과 시 자동 후속: 옵션 A (현 정책 유지 + Amendment 박제) 또는 옵션 B (사용자 명시 결정 후속)

**A/B/C 결정 분기 표준**:

| 옵션 | 의미 | 후속 |
|------|------|------|
| **A** | 현 정책 유지 + Amendment 박제 | 수치만 갱신 (예: N=3 → N=10) |
| **B** | ADR 폐기 + 신규 ADR 신설 + `Supersedes:` | 큰 변화 시 |
| **C** | 임계값 재조정 + Amendment 박제 | 정책 부분 완화/강화 |

**자동 탐지 workflow (ADR Z 패턴 첫 사례)**:
- `scripts/verify-z-pattern-health.mjs` — Node.js 3중 OR 임계값 + exit 3분류 + 자동 이슈 생성
- `.github/workflows/adr-z-pattern-health-v2.yml` — cron 월 09:00 KST + manual workflow_dispatch
- 임계값 충족 시 `gh issue create` 자동 발화 (workflow 권한)

**연쇄 사례** (astro-simulator 2026-05-16~17):
- ADR 박제 (#483 PR #490)
- Amendment 1 박제 (#476 PR #486)
- 즉시 발화 → [ADR Trigger] #487 박제
- 사용자 옵션 C 결정 → Amendment 2 박제 (#487 → #489)
- 자동 탐지 workflow 재발화 → [ADR Trigger] #495 박제

**의의**:
- ADR 가 정적 문서가 아닌 동적 거버넌스로 진화
- "결정 박제 → 발화 → 재결정 → 박제" 의 lifecycle 가시화
- 사용자 결정 의무를 시간 박제 + 옵션 매트릭스로 가속

## 관련 노트/링크

- ADR 자체: astro-simulator `docs/decisions/20260515-harness-managed-divergent-pattern.md`
- 자동 탐지 workflow: `.github/workflows/adr-z-pattern-health-v2.yml`
- [ADR Trigger] 이슈 예시: astro-simulator #487 (수동 박제, 옵션 C 결정), #495 (자동 박제, 결정 대기)
- 관련 volt: #45 (workflow_dispatch 함정), #102 (YAML on boolean), #97 (workflow_dispatch release 가속)
