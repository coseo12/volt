---
title: vsync cap이 fps bench의 함정 — 모든 시나리오 동률로 baseline 무력화
type: report
source_repo: coseo12/astro-simulator
source_issue: 11
captured_at: 2026-04-15
status: inbox
tags: [performance, benchmark, playwright, chromium, vsync]
related:
  - https://github.com/coseo12/astro-simulator/pull/158
  - https://github.com/coseo12/astro-simulator/blob/main/docs/benchmarks/p3b-perf.md
---

## 배경/상황

P3-B WebGPU 활성화 후 `bench:webgpu` 1차 측정 결과 — Newton/Barnes-Hut/WebGPU 세 엔진 모두 N=1000~10000에서 정확히 120 fps로 동률. "세 엔진 동일 성능" 결론이 나왔지만 실제로는 vsync cap(120fps Apple ProMotion)에 모두 도달한 결과였음. baseline 비교 의미가 무력화됨.

## 내용

### 발견

`bench-webgpu.mjs`의 Playwright Chromium launch args:
```
--use-angle=metal --enable-gpu-rasterization --ignore-gpu-blocklist
```

위 flag만으로는 vsync cap 해제 안 됨. measureFps 함수가 requestAnimationFrame 카운트 기반이라 vsync에 동기.

콘솔 결과:
```
newton      N=1000 : 119.96 fps
barnes-hut  N=1000 : 120.02 fps
webgpu      N=1000 : 120.05 fps
```
→ 모두 동일 = baseline 비교 무의미.

### 해결

Chromium launch args에 vsync 해제 flag 추가:
```
--disable-gpu-vsync
--disable-frame-rate-limit
--disable-renderer-backgrounding
--disable-background-timer-throttling
```

2차 측정:
```
newton      N=1000 : 868.38 fps  ← vsync 해제 효과
newton      N=10000: 220.37 fps
```

절대 throughput 가시화 → 회귀/가속 비교 가능.

### 추가 관찰

- 1차 측정에서 모든 엔진 vsync cap 도달 = N=10000에서도 GPU 한계 미달 신호
- 헤드리스 환경에서 vsync 해제는 baseline 한계 진단 + perf 회귀 측정 모두에 필수
- 단, Chromium은 background tab일 때 timer throttling 적용 → `--disable-background-timer-throttling`도 함께 필요

## 교훈 / 다음에 적용할 점

- **fps bench 작성 시 vsync 해제 flag 기본 적용**: `--disable-gpu-vsync` 반드시. CI든 로컬이든 cap 도달은 측정 무의미.
- **"모든 시나리오 동률" 신호를 의심**: 이론적으로 그럴 수 없는 환경에서 동률이면 cap 도달 가능성 우선 검토.
- **상대 비교 vs 절대 throughput 구분**: 사용자 체감(60fps 만족 여부)은 cap 측정으로 충분, 알고리즘/엔진 비교는 cap 해제 필수.
- 동일 패턴이 GPU 외에도 적용: layout thrash, animation 측정, scroll perf 등.
