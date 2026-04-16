---
title: URL 쿼리 옵트인으로 bench/진단 전용 경로 게이팅
type: knowledge
source_repo: coseo12/astro-simulator
source_issue: 18
captured_at: 2026-04-16
status: refined
tags: [pattern, feature-flag, bench, instrumentation]
related:
  - gpu-ms-vs-sim-time-pitfall.md
  - chromium-webgpu-timestamp-query-flags.md
---

## 요약

"프로덕션 사용자에게는 안전한 기본 경로, bench/개발자에게는 heavy path" 요구를 코드 분기 대신 **URL 쿼리 파라미터 옵트인**으로 구현한다. 프로덕션 회귀가 없고, bench 스크립트는 URL만 바꾸면 되며, 토글 UI도 불필요.

## 본문

**패턴**

```tsx
// 앱 부팅 시 URL param 파싱
const flagParam = new URLSearchParams(window.location.search).get('featureX');
const enabled = flagParam === '1';

// enabled 값을 scene/core 옵션으로 전달
const scene = createScene({ featureX: enabled });

// bench 전용 측정치는 window에 optional 노출
if (enabled && core.enableFeatureX()) {
  Object.defineProperty(window, '__featureXMetric', {
    configurable: true,
    get: () => core.readFeatureXMetric(),
  });
}
```

**사례**

- `?beltNbody=1` — 소행성대를 N-body 엔진에 편입 (기본 Kepler 해석해). 프로덕션은 Kepler 유지 → 회귀 없음. bench에서만 heavy path로 측정.
- `?gpuTimer=1` — GPU frame time 측정 활성화 (오버헤드 1~3%). 프로덕션 기본 off, bench/디버그에서만 on.

**장점 vs 대안**

| 방식 | 프로덕션 회귀 | 토글 UI 필요 | bench 자동화 | 코드 경로 수 |
|---|---|---|---|---|
| URL 쿼리 옵트인 | **없음** | 불필요 | URL만 수정 | 1개 (내부 분기) |
| UI 토글 | 가능 (토글 상태 관리) | 필요 | 토글 조작 필요 | 1개 |
| 빌드 플래그 (env) | 없음 | 불필요 | 빌드 재작성 필요 | 2개 |
| 기본값 변경 + 옵트아웃 | **높음** | 불필요 | 동일 | 1개 |

**적용 가이드라인**

- 기본값 = 안전한/가벼운 경로 (프로덕션 기본 경험)
- 옵트인 = heavy / 측정 / 실험 경로
- bench 스크립트에서는 URL 한 줄 변경으로 활성
- 측정 API는 `window.__name` getter로 노출 (bench는 `page.evaluate`로 폴링)
- 미지원 환경(capability 부족)에서 옵트인이 와도 silently null 반환 → 테스트 skip

## 참고

- 사례: `apps/web/src/components/sim-canvas.tsx`의 `?beltNbody=1`, `?gpuTimer=1`
- 유사 패턴: React Devtools, Chrome `?debug=1`, Next.js `?__nextDevtoolbar`
