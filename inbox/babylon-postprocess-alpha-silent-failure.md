---
title: Babylon WGSL PostProcess 알파 채널 = 0 → 캔버스 검정 (silent failure)
type: knowledge
source_repo: coseo12/astro-simulator
source_issue: 25
captured_at: 2026-04-18
status: inbox
tags: [babylon, wgsl, glsl, postprocess, webgpu, alpha-channel, silent-failure, shader-debug]
related:
  - ./wgsl-texture-sample-uniform-control-flow.md
  - ./babylon-v9-wgsl-postprocess-declarations.md
  - ../meta/patterns/adr-prime-variant-pattern.md
---

## 요약

Babylon.js PostProcess의 fragment shader가 mix/blend 결과의 알파(`.a`)를 그대로 출력할 경우, `textureSample`이 반환한 알파(종종 0)가 backbuffer compositor에 전달되어 **캔버스 전체가 검게 렌더링**된다. 콘솔/셰이더 컴파일 에러는 전혀 발생하지 않는 silent failure. 해결은 출력 직전 알파를 1.0으로 강제(`vec4f(result.rgb, 1.0)` WGSL / `vec4(result.rgb, 1.0)` GLSL)하는 1~2줄 수정.

## 본문

**증상**:

- Babylon PostProcess 진입 시 캔버스 전체 검은색
- WebGPU/WebGL2 양쪽 재현
- 콘솔 셰이더 컴파일 에러 0건
- UI/네트워크/WASM은 모두 정상 로드
- 다른 PostProcess나 기본 렌더링(scene 직접)은 정상

**원인**:

Babylon의 PostProcess 출력은 스크린-스페이스 쿼드의 fragment로, backbuffer compositor가 그 RGBA를 compositing step에서 사용. 이때:

- `textureSample(samplerName, sampler, uv)` 반환값의 `.a`가 텍스처 포맷에 따라 0이 될 수 있음 (특히 RGBA32F float 텍스처, 기본 initialization)
- mix/blend 결과:
  ```wgsl
  let result = mix(originalColor, effectColor, alpha);
  // result.a가 보간 대상 중 하나의 0 알파에 의해 0이 됨
  return result;  // ← 캔버스 검정
  ```
- compositor는 alpha=0을 "투명"으로 해석 → canvas 배경색(검정) 노출

**왜 silent인가**:

- 셰이더 컴파일/바인딩은 정상 (타입/syntax 문제 아님)
- Babylon/browser가 "알파 0인 픽셀" 자체를 에러로 간주 안 함
- 디버깅이 어려워 추정 원인을 여러 개 거친 끝에 발견 가능

**해결**:

WGSL:

```wgsl
let result = mix(originalColor, effectColor, alpha);
return vec4f(result.rgb, 1.0);  // 알파 강제
```

GLSL:

```glsl
vec4 result = mix(originalColor, effectColor, alpha);
gl_FragColor = vec4(result.rgb, 1.0);
```

**회피 패턴 (선택)**:

- 원본 텍스처 알파가 보장된다면 `result.a`를 `originalColor.a`로 교체
- 분기 출력 시 모든 분기에서 알파 1.0 명시적 보장

**재발 방지**:

- PostProcess 코드 리뷰 시 fragment shader의 출력 `.a` 채널이 항상 1.0(또는 의도된 값)으로 강제되는지 점검 체크리스트
- WGSL/GLSL 듀얼 셰이더일 때 양쪽 일관 적용

## 관련 노트/링크

- astro-simulator PR #195 (P6-B) — 본 패턴이 표면화된 PR
- astro-simulator 커밋 `a14907a` — 4줄 수정으로 해결
- P5-D(`gravitational-lensing.ts`)는 우연히 마지막 분기에 `originalColor` 전체(.a 포함)를 통과시켜 회피했던 케이스 — 같은 함정이 숨어있는 PostProcess가 더 있을 가능성
