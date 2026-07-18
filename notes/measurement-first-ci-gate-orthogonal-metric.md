---
title: 가드 측정방법론 — 구현 전 CI 게이트 + fail-fast 직교 지표 분류
type: knowledge
source_repo: coseo12/astro-simulator
source_issue: 122
captured_at: 2026-07-19
status: refined
tags: [guard-design, measurement-first, fail-fast, ci-diagnostic, cross-validate, fps, headless-rendering]
related:
  - ./measurement-first-architect-dev-handoff.md
  - ./dod-fail-measurement-verify-first.md
  - ../meta/patterns/regression-guard-3stage-simulation.md
---

## 요약

CI-only 재현 아티팩트를 다루는 가드 설계에서 검증된 두 직교 패턴. (1) **구현 전 CI 게이트**: 설계 전제가 로컬 재현 불가일 때, 구현·머지 후 무효 판명되는 낭비를 막기 위해 구현 전에 CI diagnostic dispatch로 전제를 결정적으로 판별하는 게이트를 선행한다. (2) **fail-fast 직교 지표 분류**: "측정치가 낮으면 무효 처리(skip)"는 진짜 회귀를 은폐하므로, 낮은 측정치는 분류만으로 절대 PASS시키지 않고 원인과 독립인 양성 지표가 "정상"을 능동 입증할 때만 흡수한다.

## 본문

**맥락**: `fps-baseline-guard`의 rAF 카운트 fps 측정이 CI runner(swiftshader + CPU 4x)에서 vsync 반속(30Hz) 락에 걸려 desktop만 [28,36] 대역에 밀집하는 이봉 분포 → baseline 임계 구조적 FAIL. 런타임 0줄 PR에서도 발생하는 측정 아티팩트로 margin 조정·머신 재추첨(escalation)으로 흡수 불가.

---

**패턴 1 — 로컬 무재현 전제의 "구현 전 CI 게이트" 선행 (measurement-first 상향)**

- 문제: 해법(옵션 b)이 전제 H2(락은 presentation-side, 실제 렌더는 정상)에 의존하는데, 경쟁 가설 H3(deadline-miss, 실제로 느림)이면 해법이 무효화된다. 그런데 이 판별은 **로컬 재현 불가**(macOS 120Hz는 락 미발생)라 정적 분석으로 확정 불가.
- 안티패턴: architect 초안은 "머지 후 자연 발생 시 확정"으로 절충 → **구현·머지 후에야 전제가 거짓으로 판명되는 낭비 구조**.
- 패턴: 전제 판별을 **구현 전 게이트**로 상향. 자연 발생이 확률적이면 **인위 유발**(CPU throttle 8~10x로 데드라인 미스 강제)로 게이트를 무기한 지연 없이 통과. 측정 전용 diagnostic(판정 로직 없음)을 먼저 넣고 CI dispatch로 실측 → 전제 확정 후에만 본구현.
- 실측 결과: 4x 운영 throttle에서 desktop 30Hz 락이 **자연 재현**(rafFps 33 + mobile 60 = viewport 비대칭 시그니처), 락 상태에서 vsync-우회 render-capacity가 rafFps의 **32~58배** + 정상 viewport와 동일 오더 → **H2 확정 / H3 배제**. 부수로 기존 baseline 값(중간값 49.9)이 부분 락 캡처였음도 드러남.
- cross-validate 정합: 외부 검증 모델(agy)이 원래 "measurement-first phase를 Accepted 전 게이트로" 권고했고, 사용자가 architect 절충보다 이 원권고를 채택(상향). 즉 게이트 선행은 cross-validate 원권고와 수렴.

**패턴 2 — fail-fast 직교 양성 지표 분류 (측정 무효 skip 회귀 은폐 회피)**

- 안티패턴: 아티팩트로 의심되는 낮은 측정치를 "측정 무효 → skip/pass" 처리 → 진짜 성능 회귀가 같은 낮은 값을 낼 때 **은폐**(guard-design fail-fast 위반).
- 패턴: 판정식을 `측정치 ∈ 아티팩트대역 ∧ 직교지표 ≥ 임계 → 흡수(PASS)`로 구성. 직교지표는 아티팩트 원인과 독립인 양성 신호(여기선 rAF를 우회한 동기 렌더 루프 = render-capacity). **낮은 측정치는 분류만으로 절대 PASS 안 됨** — 직교지표가 "정상"을 능동 입증할 때만 흡수. 직교지표가 저/null이면 대역 정중앙이어도 회귀 FAIL(실패 방향 폴백).
- 결정적 검증: script-level simulate hook(측정 함수 wrapper 캡슐화)으로 양방향을 CI에서 결정적 재현 — `vsync-lock`(대역+직교지표 정상 → success 흡수) / `regression`(대역+직교지표 저 → failure 차단). 로컬 무재현 아티팩트라 simulate가 PR 단계 유일 결정적 검증 경로.
- 무침범: 직교지표 프로브는 프로덕션에 이미 노출된 핸들(`window.__simCore.scene`)을 재사용 → 앱/코어 0줄. 측정 로직 강화가 앱 계약을 신설하지 않는 구조적 증거.

**운영 함정(부수 박제)**: 워크플로 concurrency group이 sha 기준이면, 같은 커밋에서 여러 diagnostic dispatch(throttle 4x/8x/10x)가 동일 group으로 묶여 **서로 취소**된다 → 진단 dispatch는 순차 실행하거나 group에 파라미터를 포함해 분리해야 함.

## 관련 노트/링크

- astro-simulator #820 / PR #821 (forensic ADR docs/decisions/20260710-820-fps-vsync-lock-forensic.md, Phase 0 게이트 + Amendment 1 H2 확정)
- 선행: astro-simulator #779 (2-job fresh-runner escalation — 본 이슈는 escalation으로 흡수 불가한 별개 클래스로 분리)
- 연관 measurement-first knowledge: volt #101 (권고→dev 실측 false-positive→정확화 3중 박제), volt #32 (수치 DoD 미달 시 측정 방법 검증 우선)
