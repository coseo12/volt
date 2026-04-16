---
title: headless Chromium WebGPU timestamp-query 활성 flag 3종 세트
type: knowledge
source_repo: coseo12/astro-simulator
source_issue: 16
captured_at: 2026-04-16
status: inbox
tags: [webgpu, chromium, playwright, benchmark, toolchain]
related:
  - gpu-ms-vs-sim-time-pitfall.md
  - url-query-opt-in-bench-gating.md
---

## 요약

headless Chromium(Playwright 등)에서 WebGPU timestamp-query resolve를 **실제 값**으로 받으려면 3개 flag 조합이 필요. 1~2개만으로는 query가 성공하고 count는 증가하지만 값이 전부 0ns로 기록되어 bench 결과가 무의미해진다.

## 본문

**필요 flag 조합 (전부 필수)**

```bash
--enable-unsafe-webgpu
--enable-webgpu-developer-features
--enable-dawn-features=allow_unsafe_apis
```

**동작 모드**

| Flag 조합 | adapter | timestamp-query feature | resolve 값 |
|---|---|---|---|
| 없음 | null | n/a | n/a |
| `--enable-unsafe-webgpu` | OK | 요청 시 거부될 수 있음 | 미확인 |
| `--enable-unsafe-webgpu` + `--enable-webgpu-developer-features` | OK | 활성 | 부분 |
| 3개 전부 | OK | 활성 | **실제 ns 값** |

**Babylon 쪽 사전 조건 (같이 필요)**

1. WebGPUEngine 생성 시 `deviceDescriptor: { requiredFeatures: ['timestamp-query'] }` (어댑터가 지원 시)
2. `EngineInstrumentation.captureGPUFrameTime = true`
3. `engine.getCaps().timerQuery`가 true여야 내부 경로 활성 — false면 조용히 비활성

**디버그 팁**

값이 0ns로만 나오면 (`count > 0`인데 `lastSecAverage == 0`) flag 부족이 거의 확실. adapter feature 리스트 확인:

```js
const adapter = await navigator.gpu.requestAdapter();
console.log(Array.from(adapter.features)); // 'timestamp-query' 포함 여부
```

**실기기 vs headless**

- 프로덕션 Chrome/Edge는 **flag 없이도** timestamp-query가 활성 (Chrome 113+)
- headless 전용 제약. CI에서 flag 누락 시 bench 결과가 조용히 오염되므로 주의

## 참고

- Chromium WebGPU flags: https://chromestatus.com/feature/6213121689518080
- Dawn features: https://dawn.googlesource.com/dawn/+/refs/heads/main/docs/dawn/features/
- 사례: `scripts/bench-webgpu.mjs` P4-D #166에서 확인
