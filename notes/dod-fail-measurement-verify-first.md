---
title: 수치 DoD 미달 시 측정 방법 검증 우선 — LRL + Newton baseline subtraction
type: knowledge
source_repo: coseo12/astro-simulator
source_issue: 32
captured_at: 2026-04-18
status: refined
tags: [physics, measurement, debugging-method, sprint-contract, numerical-verification]
related:
  - ../meta/patterns/measurement-verify-step4-data-integrity.md
  - ./ssot-json-field-sign-misread.md
---

## 요약

수치 DoD(±% 기준) 미달 시 **식을 수정하기 전에 측정 방법의 정확성을 먼저 검증**한다. 샘플링/윈도우/노이즈 특성이 DoD 미달의 진짜 원인인 경우가 많다. Astro-simulator P7-A에서 "EIH 식 structural bias 2%"로 보였던 현상이 실제로는 `min_r` 샘플링이 특정 방향으로 pull되는 노이즈였음이 LRL 벡터 + Newton baseline subtraction으로 드러남.

## 본문

**상황**: P6-D에서 Velocity-Verlet로 지구 근일점 세차 측정 시 rel_err 2.48% 기록. P7-A에서 Yoshida 4차 적분기 격상 후 "더 좋아져야 할" rel_err이 1.87%로 **여전히 DoD(<1.25%) 미달**. `dt ∈ {10, 30, 60}s` 변화에도 동일값 수렴 — 적분기 한계 아님.

**Single1PN(Schwarzschild 정확해) 대조 실험**:

| 행성 | Single1PN 1c | EIH1PN 1c |
|---|---|---|
| 수성 | 42.93″ (0.11%) | 42.93″ (0.11%) |
| 지구 | 3.77″ (1.86%) | 3.77″ (1.87%) |
| 금성 | 8.41″ (2.42%) | 8.41″ (2.42%) |

Single1PN(시험입자 근사의 정확한 구현)과 EIH1PN이 **동일 deviation** → 식 문제 아니라 측정 문제.

**측정법 전환**:
1. **LRL 벡터** (`A = v × L - μ r̂`) 로 근일점 방향을 analytical하게 직접 계산 — 저이심률(지구 e=0.017)에서도 정확
2. **Newton baseline subtraction** — 동일 setup Newton 적분으로 수치 세차를 측정 후 EIH 결과에서 빼서 물리 세차만 isolate

**centuries 수렴 확인**: 금성 1c 2.42% → 10c 1.39%, 지구 1c 1.87% → 3c 1.19% → 10c 0.85%. 측정 윈도우를 늘리면 이론값 수렴 확인.

**결론**: P6-D의 0.63% (금성) 역시 측정 노이즈의 우연한 이론값 일치였음. Yoshida4가 더 정밀/재현성 있음이 측정법 전환으로 비로소 드러남.

**키 교훈**: 
- DoD 실패 시 (a) 식 수정 → (b) 알고리즘 교체 순이 자동 반사. 실제로는 (0) 측정 방법 검증이 0번 단계여야 함.
- 교과서 이론값을 기준으로 하더라도, 측정 대상 신호가 작을 때(지구 GR 세차 3.84″ ≪ 수성 42.98″) 샘플링 노이즈가 이론값 방향으로 우연히 pull될 수 있음.
- 단일 century 적분의 잔차는 저이심률 궤도에서 특히 큼 — secular 수렴을 위해 다중 centuries 필요.

## 관련 노트/링크

- ADR: https://github.com/coseo12/astro-simulator/blob/main/docs/decisions/20260418-p7-integrator-upgrade.md
- P7-A PR: https://github.com/coseo12/astro-simulator/pull/212
- Phase C 진단 코멘트: https://github.com/coseo12/astro-simulator/issues/206#issuecomment-4273116871
- Yoshida H. (1990) Phys. Lett. A 150, 262–268 (적분기 원출처)
