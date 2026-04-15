---
title: GPU 셰이더 테스트의 CPU mirror 패턴 — node 환경에서 알고리즘 정합성 가드
type: knowledge
source_repo: coseo12/astro-simulator
source_issue: 10
captured_at: 2026-04-15
status: refined
tags: [testing, gpu, webgpu, wgsl, ci]
related:
  - https://github.com/coseo12/astro-simulator/pull/149
  - https://github.com/coseo12/astro-simulator/pull/150
  - https://github.com/coseo12/astro-simulator/pull/152
---

## 요약

WGSL/HLSL/Metal 셰이더는 node 환경에 GPU가 없어 단위 테스트 불가. 동일 알고리즘을 f32 CPU(plain JS/TS)로 미러 구현해 알고리즘 정합성을 vitest로 가드한다.

## 본문

### 배경

GPU 셰이더 코드는 보통 다음 단계로 검증한다:
- (a) CI에서 GPU 활성 환경 마련 — 어렵고 비쌈 (lavapipe/dawn 셋업)
- (b) 브라우저 E2E (Playwright) — 무거움, 환경별 가용성 가변
- (c) 수동 검증 — 회귀 못 잡음

→ 가벼운 단위 테스트가 부재.

### 패턴

GPU 셰이더와 **동일 알고리즘**을 CPU에서 f32로 구현. 실 GPU 결과 ≈ CPU mirror 결과로 가정하고, mirror 함수의 정합성을 vitest로 검증.

```ts
// gpu/nbody-force-cpu.ts — WGSL과 동일한 direct sum, f32 누적
export function computeForcesF32(positions: Float32Array, masses: Float32Array,
  softeningSq: number, G: number): Float32Array {
  const n = masses.length;
  const acc = new Float32Array(3 * n);
  for (let i = 0; i < n; i++) {
    for (let j = 0; j < n; j++) {
      if (j === i) continue;
      // ... WGSL의 _bh_pair_acc와 동일 수식
    }
  }
  return acc;
}
```

검증 항목:
- 단순 케이스(2-body 대칭, 단일 입자 self-force=0)
- softening 발산 방지
- 상한 (모든 위치 finite, 발산 없음, 모멘텀 보존)

### 한계와 보완

- f32 정밀도까지만 정합. f64 reference(예: `NBodyEngine`)와는 별도 비교
- 실제 GPU 정확도(드라이버별 fma 차이 등)는 별도 e2e 필요 — but "알고리즘 자체가 맞는지"는 mirror로 확정
- 새 셰이더 추가 시 mirror 함수 의무화 → 회고에서 가드로 박제

### 본 프로젝트 적용

- `packages/core/src/gpu/nbody-force-cpu.ts` (WGSL force shader mirror)
- `packages/core/src/gpu/nbody-vv-cpu.ts` (WGSL V-V integrator mirror)
- vitest 가드: 153/153 통과, 새 셰이더 추가 시 mirror 동시 작성
