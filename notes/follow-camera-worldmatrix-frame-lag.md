---
title: 3D 엔진 follow-camera 한 프레임 lag — onBeforeRender 시점 worldMatrix 미갱신 패턴
type: knowledge
source_repo: coseo12/astro-simulator (이슈 #611 / PR #612)
source_issue: 119
captured_at: 2026-07-09
status: refined
tags: [babylonjs, 3d-rendering, camera, render-loop, timing-bug, worldMatrix, follow-camera, frame-lag]
related:
  - ./babylon-postprocess-alpha-silent-failure.md
  - ./babylon-v9-wgsl-postprocess-declarations.md
  - ./downstream-measurement-final-guard.md
---

## 요약

매 프레임 객체를 추적하는 follow-camera(`camera.target = mesh.absolutePosition`)가 `onBeforeRender` 콜백에서 worldMatrix 갱신 없이 `absolutePosition`을 읽으면 **항상 직전 프레임 위치**를 반환해 target이 한 프레임 뒤처진다. lag = `dt × 객체 각속도`이므로 **빠르게 움직이는 객체(위성처럼 각속도 큰)에서만 가시적 회귀로 발현**하고 느린 객체는 tolerance 내라 잠복한다. 진단 핵심은 "측정 코드는 `computeWorldMatrix(true)` 후 읽는데 추적 코드는 미갱신으로 읽는 **비대칭**"을 찾는 것.

## 본문

**메커니즘 (Babylon.js 기준, 일반 3D 엔진 공통):**

render loop 순서가 `physics/position update → scene.render()` 일 때:
1. 프레임 N: 위치 업데이트가 `mesh.position = P(N)` 설정 → worldMatrix dirty
2. `scene.render()` → `onBeforeRenderObservable` 발동 → follow 콜백이 `mesh.absolutePosition` 읽음
3. 이 시점 worldMatrix는 **아직 이번 프레임 미계산** → `absolutePosition`은 직전 프레임 worldMatrix 계산값 `P(N-1)` 반환 (Babylon `absolutePosition`은 `_absolutePosition` 캐시 필드로, `mesh.position` set만으론 자동 갱신 안 됨)
4. follow가 `camera.target = P(N-1)` → 렌더 중 worldMatrix 계산되어 `_absolutePosition = P(N)`

→ target이 항상 한 프레임 뒤처짐. `targetΔ = |P(N) - P(N-1)|` = 한 프레임 이동량.

**왜 일부 객체만 발현하나 (잠복 함정):**

lag 크기 = `dt × 각속도`. 회귀 가드 tolerance를 초과하는지가 객체별로 갈린다.
- 빠른 객체(위성: 궤도 주기 7~30시간 → 프레임당 큰 각도 이동) → tolerance 초과 → FAIL
- 느린 객체(행성: 주기 수십~수백 일) → tolerance 내 → PASS (잠복)
- 같은 종류라도 주기 짧을수록 강발현
- 무거운 프레임(panel resize 등 dt↑)에서 이동량 증폭 → run별 flaky 유발

**진단 패턴 (재사용 가능):**

"측정/검증 코드"와 "런타임 추적 코드"가 같은 값을 다루는데 결과가 다르면, **둘의 worldMatrix 갱신 시점 비대칭**을 의심하라. 측정은 보통 `computeWorldMatrix(true)` 후 읽도록 작성되지만(정확성 우선), 런타임 추적은 성능 이유로 생략하는 경우가 많다.

**Fix:**

follow 콜백에서 `mesh.computeWorldMatrix(true)` 후 `absolutePosition` 읽기 (1라인). focus/추적 대상 1개만 매 프레임 재계산이라 비용 무시 가능. 대안(추적 콜백을 `onAfterRender`로 이동, 위치 업데이트 직후 worldMatrix 강제 갱신)보다 국소적이고 측정 코드와 시점이 정합한다.

**주의 — parent 체인:** 추적 대상이 다른 노드의 자식이면 `computeWorldMatrix(true)`가 dirty parent까지 갱신하는지 확인. 단, 절대좌표를 직접 기록하는 독립 mesh(parent 없음)면 자기 matrix 재계산만으로 충분.

## 관련 노트/링크

- astro-simulator #611 / PR #612 (camera-controller follow observer worldMatrix 갱신)
- 일반화: "측정 통과 ≠ 런타임 동작" — 검증 코드와 프로덕션 코드의 상태 갱신 시점 비대칭이 잠복 버그 생성원
- 잠복 함정 변형: tolerance 기반 회귀 가드는 "약발현 객체"를 통과시켜 회귀를 무한 잠복시킬 수 있음 (강발현 객체가 가드에서 제외/skip되면 영구 미검출)
