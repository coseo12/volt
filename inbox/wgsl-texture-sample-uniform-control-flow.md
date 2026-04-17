---
title: WGSL textureSample은 uniform control flow에서만 호출 가능 — branchless 패턴
type: knowledge
source_repo: coseo12/astro-simulator
source_issue: 19
captured_at: 2026-04-17
status: inbox
tags: [wgsl, webgpu, shader, postprocess, pitfall]
related:
  - ./babylon-v9-wgsl-postprocess-declarations.md
---

## 요약

WGSL에서 `textureSample()`은 non-uniform control flow(if/else 분기) 안에서 호출할 수 없다. GPU 파이프라인의 파생(derivative) 연산이 non-uniform 분기에서 정의되지 않기 때문. GLSL에서는 `texture2D`가 자유롭게 사용 가능하지만 WGSL로 전환 시 반드시 branchless 패턴으로 리팩토링해야 한다.

## 본문

**증상**

```
Error while parsing WGSL: 'textureSample' must only be called from uniform control flow
```

Babylon WebGPU PostProcess에서 WGSL shader compilation 실패 → 검은 화면 (에러는 console에만 표시, 크래시 없이 fallback).

**원인**

```wgsl
// ❌ non-uniform control flow — 컴파일 에러
if (dist > threshold) {
  fragmentOutputs.color = textureSample(tex, samp, uv);
  return fragmentOutputs;
}
var color = textureSample(tex, samp, deflectedUV); // 여기도 에러
```

`dist`는 픽셀마다 다른 값이므로 non-uniform. GPU의 warp/wavefront 내에서 분기가 발생하면 derivative 연산(textureSample 내부의 mipmap LOD 계산)이 정의되지 않는다.

**해결: branchless 패턴**

```wgsl
// ✅ uniform control flow — 분기 밖에서 textureSample 1~2회 호출
let originalColor = textureSample(tex, samp, uv);
let lensedColor = textureSample(tex, samp, deflectedUV);

// step()/mix()로 분기 없이 결과 조합
let outside = step(threshold, dist);
let inside = step(dist, innerThreshold);
let result = mix(
  mix(lensedColor, vec4f(0.0, 0.0, 0.0, 1.0), inside),
  originalColor,
  outside
);
fragmentOutputs.color = result;
```

**장점**: branchless는 성능에도 유리 (GPU warp divergence 없음).
**단점**: 가독성 하락. 주석으로 의도를 보충해야 함.

**GLSL과의 차이**

| 항목 | GLSL | WGSL |
|---|---|---|
| `texture2D` in if | ✅ 허용 (undefined behavior 가능하지만 컴파일됨) | ❌ 컴파일 에러 |
| derivative 보장 | 프로그래머 책임 | 언어 수준 강제 |
| 해결 패턴 | 필요 없음 | `step()/mix()` branchless |

## 관련 노트/링크

- WGSL spec: https://www.w3.org/TR/WGSL/#uniformity
- 사례: `packages/core/src/scene/gravitational-lensing.ts` P5-D #180
- `textureSampleLevel()`은 LOD를 명시하므로 non-uniform 분기에서도 사용 가능 (대안)
