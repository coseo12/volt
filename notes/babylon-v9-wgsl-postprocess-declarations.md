---
title: Babylon v9 WGSL PostProcess shader 선언 규약 (미문서화)
type: knowledge
source_repo: coseo12/astro-simulator
source_issue: 20
captured_at: 2026-04-17
status: refined
tags: [babylon, webgpu, wgsl, postprocess, undocumented]
related:
  - ./wgsl-texture-sample-uniform-control-flow.md
  - ./babylon-postprocess-alpha-silent-failure.md
---

## 요약

Babylon.js v9에서 WGSL PostProcess fragment shader를 작성할 때 `varying vUV`, `var textureSamplerSampler`, `var textureSampler` 선언이 **필수**이며 순서도 고정이다. 공식 문서에 미기재 — 내장 pass shader(`ShadersWGSL/pass.fragment.js`) 역공학으로 발견.

## 본문

**Babylon WGSL PostProcess 선언 템플릿**

```wgsl
varying vUV: vec2f;
var textureSamplerSampler: sampler;
var textureSampler: texture_2d<f32>;
// 사용자 custom uniform 추가
uniform myParam: f32;

@fragment
fn main(input: FragmentInputs) -> FragmentOutputs {
  let color = textureSample(textureSampler, textureSamplerSampler, input.vUV);
  fragmentOutputs.color = color;
  return fragmentOutputs;
}
```

**필수 요소**

| 선언 | 역할 | 누락 시 |
|---|---|---|
| `varying vUV: vec2f` | UV 좌표 입력 | `input.vUV` 접근 불가 또는 검은 화면 |
| `var textureSamplerSampler: sampler` | 텍스처 샘플러 | textureSample 바인딩 실패 |
| `var textureSampler: texture_2d<f32>` | 입력 텍스처 | 검은 화면 (바인딩 없음) |

**GLSL과의 차이**

- GLSL PostProcess: Babylon이 `precision`, `varying vUV`, `uniform sampler2D textureSampler`를 **자동 삽입**
- WGSL PostProcess: **사용자가 직접 선언해야 함**. Babylon이 자동 삽입하지 않음.

**ShaderStore 등록**

```ts
import { ShaderStore } from '@babylonjs/core/Engines/shaderStore.js';
import { ShaderLanguage } from '@babylonjs/core/Materials/shaderLanguage.js';

ShaderStore.ShadersStoreWGSL["myEffectFragmentShader"] = wgslCode;

const pp = new PostProcess("myEffect", "myEffect", uniforms, null, 1.0, camera,
  undefined, undefined, undefined, undefined, undefined, undefined, undefined,
  undefined, undefined, ShaderLanguage.WGSL);
```

**발견 경로**

1. GLSL shader 작성 → WebGPU에서 검은 화면
2. `varying vUV` 수동 선언 제거 → UV 테스트는 성공하지만 textureSample 검은색
3. 내장 pass shader 역공학: `ShadersWGSL/pass.fragment.js`
4. `varying vUV: vec2f` 선언 + 순서 일치 후 정상 동작

## 관련 노트/링크

- 내장 pass shader: `@babylonjs/core/ShadersWGSL/pass.fragment.js`
- Babylon PostProcess 공식 문서: https://doc.babylonjs.com/features/featuresDeepDive/postProcesses/usePostProcesses (WGSL 예시 미포함)
- `Effect.ShadersStore` (GLSL) vs `ShaderStore.ShadersStoreWGSL` (WGSL) — import 경로도 다름
