---
title: Velocity-Verlet 시간 역행 대칭성 보장 조건
type: knowledge
source_repo: coseo12/astro-simulator
source_issue: 6
captured_at: 2026-04-14
status: refined
tags: [physics, symplectic, nbody, numerical]
related:
  - https://en.wikipedia.org/wiki/Verlet_integration
  - https://en.wikipedia.org/wiki/Symplectic_integrator
---

## 요약

Velocity-Verlet(Leapfrog) 심플렉틱 적분기는 특정 조건을 만족할 때 전진 +T 후 역행 -T로 초기 상태를 부동소수점 rounding 수준으로 복원한다. 조건을 알면 시뮬레이터의 "시간 역행" UX를 물리적으로 정확하게 구현 가능.

## 본문

**보장 조건(3가지)**:
1. **고정 dt**: 전진 구간과 역행 구간의 서브스텝 dt 절댓값이 동일해야 함. `step_chunked(total, max_dt)`에서 `sub_count = ceil(|total|/max_dt), sub_dt = total/sub_count`로 균등 분할 시 +T와 -T에서 sub_count가 같고 부호만 반전 → 조건 충족.
2. **힘 = 위치만의 함수**: 가속도가 위치에만 의존하고 속도/외력과 독립. 중력(G·m/r²·r̂)은 조건 만족.
3. **동일한 N·질량**: 역행 중에 바디 추가/삭제/질량 변경 금지.

**파괴 조건**:
- 가변 dt (frame-time 기반 서브스텝 증감) → 대칭성 파괴.
- 역행 중 바디 추가/삭제 (P2-B 소천체 동적 생성 시 주의).
- 감쇠·소산 항 추가 (공기저항·조석 등은 본질적으로 비가역).
- GPU Compute 비결정적 reduction 순서 → bit-exact 복원 실패. 결정적 reduction 강제 필요.

**실측 (태양계 9체 기준)**:
- ±10일, dt=1h: pos/vel 상대 오차 < 1e-9
- ±1년, dt=10min: pos/vel 상대 오차 < 1e-9
- 분할 왕복 5×2일: < 1e-9

**시뮬레이터 UX 함의**:
- "시간 역행" 버튼을 제공할 경우, 고정 dt를 강제해야 사용자가 되돌아간 위치가 원래 위치와 같음.
- 사용자가 파라미터(질량 등)를 변경한 시점부터는 역행으로 원 상태 복원 불가 → "리셋" UI 경로 별도 제공.
- 감쇠 효과가 필요한 확장(조석 모델 등)에서는 역행 모드를 비활성화해야 함.
