---
title: Capability 폴백 + HUD notice 패턴 — progressive feature 안전 노출
type: knowledge
source_repo: coseo12/astro-simulator
source_issue: 9
captured_at: 2026-04-15
status: refined
tags: [ux, progressive-enhancement, webgpu, fallback, capability-detection]
related:
  - https://github.com/coseo12/astro-simulator/pull/151
  - https://github.com/coseo12/astro-simulator/pull/146
---

## 요약

브라우저별 capability(WebGPU, WebGL2, ServiceWorker, Push, OffscreenCanvas 등)가 다른 progressive feature를 UI 토글로 노출할 때, 미지원 환경에서도 깨지지 않게 자동 폴백 + 사용자에게 사유를 한 줄로 안내하는 패턴.

## 본문

### 문제

새 기능(예: WebGPU N-body 엔진)을 5-mode 토글에 추가하면:

- 미지원 환경에서 토글 클릭 시 throw → 화면 깨짐
- 토글을 disabled 처리하면 사용자가 "왜 안 되는지" 모름
- URL 파라미터(`?engine=webgpu`)로 직접 진입한 사용자도 똑같이 깨짐

### 패턴

3-layer 분기:

1. **Capability detection 모듈** (단일 진입점)
   ```ts
   export async function detectCapability(): Promise<{ ok: boolean; reason?: string }>
   ```
   환경 부재 / API throw / 각 케이스 명확한 reason 반환.

2. **Resolver 함수** (UI 입력 → 실제 사용 모드)
   ```ts
   const resolveEngine = (requested) => {
     if (requested === 'webgpu' && !capability.ok) {
       store.setEngineNotice(`WebGPU 미지원 — ${capability.reason} · ${fallback}로 폴백.`);
       return fallback;
     }
     return requested;
   };
   ```
   사용자 의도(store.physicsEngine)는 보존, 실제 동작은 폴백.

3. **HUD notice** (dismissible + role=status)
   ```tsx
   {notice && (
     <div role="status" aria-live="polite">
       {notice}
       <button aria-label="알림 닫기" onClick={() => setNotice(null)}>×</button>
     </div>
   )}
   ```

### 운용

- 토글에는 모든 mode를 활성화로 노출 (사용자 의도 차단 안 함)
- URL 파라미터 후방호환 자동
- 정식 서비스에선 GA/Sentry로 폴백 발생률 추적

### 본 프로젝트 적용

- `packages/core/src/gpu/capability.ts` — `detectGpuCapability()`
- `apps/web/src/components/sim-canvas.tsx` — `resolveEngine()` 사용자 의도 vs 실제 분기
- `apps/web/src/components/layout/hud-corners.tsx` — dismissible notice
- 5-mode toggle: webgpu/auto 모두 활성, capability 미달 시 barnes-hut 자동 폴백
