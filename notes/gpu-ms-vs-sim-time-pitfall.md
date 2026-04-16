---
title: GPU ms ≠ 시뮬 시간 — CPU 시뮬 엔진 벤치마크 오독 함정
type: knowledge
source_repo: coseo12/astro-simulator
source_issue: 15
captured_at: 2026-04-16
status: refined
tags: [benchmark, webgpu, babylon, pitfall]
related:
  - chromium-webgpu-timestamp-query-flags.md
  - url-query-opt-in-bench-gating.md
---

## 요약

Babylon `EngineInstrumentation.gpuFrameTimeCounter`로 측정하는 GPU 시간은 **"GPU가 바쁜 시간"만** 집계한다. 시뮬 로직이 CPU(wasm, JS 등)에서 돌면 그 시간은 GPU ms에 안 잡히고 렌더 파트만 잡힌다. 이걸 WebGPU compute(시뮬까지 GPU에서)와 GPU ms로 1:1 비교하면 "WebGPU가 더 느림" 오독 발생. **fps 비율이 실질 기준**.

## 본문

**증상**

- Barnes-Hut(CPU wasm) GPU ms = 0.07ms
- WebGPU compute GPU ms = 2.18ms
- 순진 비교: "WebGPU가 barnes-hut보다 31× 느림" → **오독**

**실제 상황**

| 엔진 | 시뮬 위치 | 렌더 | GPU ms 의미 | fps |
|---|---|---|---|---|
| Barnes-Hut | CPU (wasm) | GPU | 렌더만 | 3.24 (N=5000) |
| WebGPU | GPU (compute) | GPU | 시뮬+렌더 | 732.95 (N=5000) |

→ **fps 비율 = 226× (WebGPU 우위)**. CPU bottleneck이 전체 throughput을 결정하므로 fps가 정직한 기준.

**원인**

- `EngineInstrumentation`은 Babylon 엔진이 issue한 `GPUQuerySet` 타임스탬프를 집계
- CPU에서 돌아간 wasm 연산은 timestamp 대상 아님
- CPU가 시뮬 500ms 쓰고 렌더에 GPU 0.07ms 썼다면 "GPU 시간"은 0.07ms가 맞지만 **실질 throughput은 2 fps**

**방어 방법**

1. bench 스크립트에 GPU ms 컬럼 주석 명시: "참고 정보, CPU 시뮬 엔진은 렌더만 측정됨"
2. 주 비교 기준은 **fps** 또는 **전체 frame time**(performance.now 기반)
3. GPU compute 자체의 순수 비용을 보려면 `ComputeShader.gpuTimeInFrame: WebGPUPerfCounter` (Babylon v9 전용) 사용 — compute dispatch 단독 측정

## 참고

- Babylon `EngineInstrumentation` d.ts: `@babylonjs/core/Instrumentation/engineInstrumentation.d.ts`
- Babylon `ComputeShader.gpuTimeInFrame`: `@babylonjs/core/Compute/computeShader.d.ts:84`
