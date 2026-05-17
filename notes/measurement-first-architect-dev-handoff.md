---
title: measurement-first 조정 패턴 — architect 권고 → dev D1 실측 false-positive → 정확화 3중 박제
type: knowledge
source_repo: coseo12/astro-simulator
source_issue: 101
captured_at: 2026-05-17
status: refined
tags: [measurement-first, architect-dev-handoff, false-positive, multi-or, identifier-precision]
related:
  - ./dod-fail-measurement-verify-first.md
  - ../meta/patterns/measurement-verify-step4-data-integrity.md
  - ./debug-script-over-static-analysis.md
---

## 요약

architect 가 권고한 검증 식별자 (multi-OR keyword 매칭 등) 가 **dev 단계 첫 실측에서 false-positive 5건 발생** → dev 가 정확 식별자 (3중 박제) 로 정정. architect 결정만으로 박제하지 말고 dev 가 D1 실측으로 검증 후 정확화하는 measurement-first 조정 패턴.

## 본문

**사례 (PR #490 ADR Z 패턴 자동 탐지, 2026-05-16)**:

architect 권고 (multi-OR keyword):
```javascript
const PHASE1_KEYWORDS = /(Phase 1|본 프로젝트|astro-simulator)/i;
const PHASE2_KEYWORDS = /(Phase 2|upstream|harness-setting)/i;
```

dev D1 실측 (`bash scripts/verify-z-pattern-health.mjs`):
```
Phase 1: 11
Phase 2: 5  ← false-positive (실제 0)
```

검증: Phase 2 매칭된 5건 라인 확인 → "upstream" 키워드가 docstring/주석에서 매칭됨. 실제 Phase 2 PR citation 0건.

dev 정정 (정확 식별자):
```javascript
const PHASE1_PR_PATTERN = /Phase 1[\s\S]{0,80}?#(\d+)/g;
const PHASE2_PR_PATTERN = /Phase 2[\s\S]{0,80}?#(\d+)/g;
// "Phase X" 직후 80 글자 이내의 #N 만 매칭 → docstring noise 제외
```

dev D1 재실측:
```
Phase 1: 11
Phase 2: 0  ← 정확
```

**3중 박제 위치** (정확 식별자):
1. `scripts/verify-z-pattern-health.mjs` 본문 (정확 정규식)
2. `.github/workflows/adr-z-pattern-health-v2.yml` step 주석 (의미 박제)
3. ADR `20260515-harness-managed-divergent-pattern.md` §측정 지표 (스펙 박제)

**패턴 일반화**:

| 단계 | 권고 | 측정 |
|------|------|------|
| architect | broad keyword OR | 자체 측정 안 함 — 권고만 |
| dev D1 | broad 적용 → 실측 | **false-positive 발견 시 정정 의무** |
| dev D2 | precision 정정 → 재실측 | 0 false-positive 확인 후 박제 |
| reviewer | 정적 검토 | precision 정규식 의미론적 검증 |
| qa | 동적 매트릭스 | 격리 케이스로 false-positive/negative 확인 |

**핵심 교훈**:
- architect 권고를 dev 가 **broad 그대로 박제 금지** — D1 실측 의무
- false-positive 발견 시 dev 가 정확화 (정규식 boundary / lookahead / character class 등)
- 정정 후 3중 박제 위치 (스크립트 본문 + workflow 주석 + ADR §측정 지표) 동시 갱신
- "measurement-first" 의 핵심: 권고는 가설, 실측이 확정

**관련 패턴**:
- harness real-lessons "DoD 수치 미달 시 측정 방법 검증 우선" (volt #32, #53) — 본 패턴은 가드 설계 단계에서의 측정 검증
- 신호가 약할 때 (측정 대상 ≪ baseline) noise 가 이론값 방향으로 pull 되어 우연 성공 기록 → measurement-first 정정 우선

## 관련 노트/링크

- 사례: PR #490 (#483 ADR Z 패턴 자동 탐지, 2026-05-16)
- 정확 식별자 박제: `scripts/verify-z-pattern-health.mjs` + workflow yaml + ADR Amendment 3
- 관련 volt: #32 (지구 GR 세차 측정 식 정정), #53 (측정법 검증 4단계 확장), #67 (debug 스크립트 실측)
