---
title: Turbopack namespace re-export 도 SSR 평가 chain 트리거
type: knowledge
source_repo: coseo12/astro-simulator
source_issue: 80
captured_at: 2026-05-04
status: refined
tags: [turbopack, nextjs, wasm, ssr, monorepo, module-graph, wasm-pack, pnpm-workspace]
related:
  - ./monorepo-core-dist-stale.md
  - ./reviewer-qa-gate-skip-headless-self-pass.md
---

## 요약

Next.js Turbopack 환경에서 monorepo core 패키지의 **sub-path export 추가** (`packages/core/package.json` exports field) 뿐만 아니라 **main entry 의 namespace re-export** (`scene/index.ts → core/src/index.ts`) 도 turbopack module dep graph 평가 시 re-export chain 을 따라가 wasm-pack `--target nodejs` 출력의 `__dirname` resolve 실패로 SSR 500 회귀를 트리거한다. 진짜 wasm-safe 패턴은 **main entry 에 named export 직접 박제 + scene/index.ts 에서 의도적 회피** (re-export 안 함).

## 본문

#### 회귀 메커니즘 (3자 결합)

```
wasm-pack --target nodejs
    ↓ (생성된 physics_wasm.js:367 의 ${__dirname} ENOENT 발현)
turbopack module dep graph evaluation
    ↓ (sub-path import OR namespace re-export 둘 다 chain 평가 트리거)
Next.js SSR `/ko` HTTP 500
    ↓ (클라이언트 hydration 부분 실패 → physics engine 미초기화)
시각 회귀 (focus 시 허공 + 반복 줌 등)
```

#### 라운드별 패턴 매트릭스

| 패턴 | SSR 결과 | 비고 |
|---|---|---|
| (라운드 1) `packages/core/package.json` exports field 에 `./scene/r-phase-allowlist` sub-path 추가 + 직접 import | 500 ❌ | turbopack 이 sub-path entry 평가 시 chain 트리거 |
| (라운드 2 초안) main entry namespace re-export (`scene/index.ts` → `core/src/index.ts`) + namespace 경유 import (`scene as sceneApi.X`) | 500 ❌ | namespace 경유도 동일 chain 평가. **라운드 2 정정 진짜 발견** |
| **(라운드 2 정정) `core/src/index.ts` 에 named export 직접 박제 + `scene/index.ts` 에서 re-export 의도적 회피 + named import 직접** | 200 ✓ | turbopack 이 명시 named export 만 평가, scene/index.ts chain 평가 회피 |

#### SSR 평가 컨텍스트 매트릭스

| 컴포넌트 | 진입 경로 | SSR 평가 | wasm chain 영향 |
|---|---|---|---|
| `sim-canvas` | `next/dynamic({ssr:false})` | 회피 | 0 |
| `focus-quick-buttons` | app-shell 직접 import | 평가 대상 | 직접 영향 |

→ SSR 평가 회피 패턴 (`next/dynamic({ssr:false})`) 가 적용된 컴포넌트만 사용하면 회귀 표면화되지 않으나, app-shell 직접 import 컴포넌트 1개라도 chain 위에 있으면 회귀 발현.

#### 회피 패턴 SSoT

```ts
// packages/core/src/index.ts (main entry)
export { isRPhaseFocusable, R_PHASE_BODY_ALLOWLIST } from './scene/r-phase-allowlist.js';

// packages/core/src/scene/index.ts
// 의도적으로 re-export 안 함 (turbopack chain evaluation 차단)
// export * from './r-phase-allowlist.js';  // ← 금지

// apps/web 호출 측
import { isRPhaseFocusable } from '@astro-simulator/core';
// import { scene as sceneApi } from '@astro-simulator/core'; // ← 금지 (namespace 경유)
```

#### 자동화 가드

`scripts/verify-core-exports-immutable.sh` — `packages/core/package.json` exports field 의 WASM 도메인 (`scene/physics/render/gpu`) sub-path 추가 시 exit 1. 화이트리스트: `. / ./coords / ./physics / ./scene / ./gpu / ./ephemeris` (순수 데이터 도메인 자유).

#### 시도/실패 fix

- `next.config.mjs` `serverExternalPackages: ['@astro-simulator/physics-wasm', '@astro-simulator/core']` — **무효** (Turbopack 이 옵션 무시 알려진 패턴)

#### 적용 대상

monorepo (pnpm workspace 등) + Next.js Turbopack + wasm-pack `--target nodejs` 출력 패키지를 transitive dep 으로 가지는 패키지의 module graph 변경 시 모두 적용. R-Phase 정합성 가드처럼 web/app-shell 직접 import 영역으로 확장되는 신규 helper 박제 작업에서 회귀 가능성 가장 높음.

## 관련 노트/링크

- ADR (라운드 1 결정 + 라운드 2 Amendment): https://github.com/coseo12/astro-simulator/blob/develop/docs/decisions/20260504-r-phase-allowlist-guard.md
- 라운드 1 회귀 PR (closed): https://github.com/coseo12/astro-simulator/pull/407
- 라운드 2 ADR Amendment (merged 4975846): https://github.com/coseo12/astro-simulator/pull/413
- 라운드 2 코드 구현 (merged d2f6adf): https://github.com/coseo12/astro-simulator/pull/414
- 자동화 가드 스크립트: `scripts/verify-core-exports-immutable.sh`
- 직접 SSoT: `packages/core/src/index.ts` (named export) + `packages/core/src/scene/index.ts` (의도적 회피)
- 관련 volt 교훈: #70 (monorepo dist stale), #77 (headless ≠ 실 Chrome — 본 회귀 표면화 강제 도구)
