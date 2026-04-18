---
title: swiftshader headless WebGPU 픽셀 freeze — 실 Chrome 수동 검증 필수
type: report
source_repo: coseo12/astro-simulator
source_issue: 33
captured_at: 2026-04-18
status: refined
tags: [webgpu, playwright, headless, chrome, manual-verification, swiftshader, ui-testing, false-positive]
related:
  - ../meta/feedback/ui-browser-verify-miss.md
  - ../meta/patterns/browser-verify-script-pattern.md
  - ../meta/patterns/adr-prime-variant-pattern.md
---

## 배경/상황

P7-C "트랙 B 3D ray construction 복원" 5단계 순차 재시도에서 3차(Frustum Corner Interpolation)가 **headless Chromium (`--use-webgpu-adapter=swiftshader`)** 에서 컴파일 성공 + 비검정 canvas 렌더 + fps=23 유지로 **"채택" 판정**. QA 단계 browser-verify-*.mjs 8/8 PASS. 이후 실 Chrome GUI 수동 검증에서 **accretion disk 렌더 실패** (블랙홀만 검은 원, disk mask 계산 실패) 확인.

## 내용

**1. swiftshader headless 한계 실측**:
- 카메라 beta 변경 후 픽셀 업데이트 freeze — `disk 장축이 화면 x축에서 벗어나는가` 같은 카메라 회전 응답 픽셀 diff 검증 불가
- `use-webgpu-adapter=swiftshader` adapter가 초기 렌더는 정상이나 camera 조작 후 frame 생성 실패/지연
- 두 번째 `page.goto()` 에서 WebGPU 컨텍스트 재초기화 실패 (`A fatal error occurred during WebGPU creation/initialization`)

**2. 컴파일 성공 ≠ 기능 성공**:
- 3차 E: 셰이더 컴파일 PASS, lensing 왜곡(background 별빛 휨) 정상 동작 — **한 가지 pipeline은 성공**
- 같은 셰이더에서 accretion disk mask 계산(`inRing3d = step(diskInner, rEcc3d) * ...`)은 실패 — disk가 렌더되지 않음
- headless 로그/스크립트 assertion만으로는 "일부 성공/일부 실패"가 혼재된 케이스를 구분 못함

**3. 실 Chrome 검증으로 드러난 것**:
- 오렌지/노랑/빨강 accretion disk 타원이 완전히 invisible
- 기본 경로 `?bh=2` (5차 D' 보강)에서는 disk 뚜렷
- 즉 3차 E의 disk mask 로직 자체의 버그 (rEcc3d / valid3d / escapeOutcome 중 하나 이상의 경로 실패)

**4. PM 계약 M1 백업 경로 활성화**:
- 3차 E를 `?ray3d=1` 실험적 경로로 보존 (lensing 성공 자산)
- 5차 D' 보강으로 전환 — `diskAxisX/Y` 를 world disk major axis의 화면 투영 방향으로 대체
- ADR 상태: `Accepted as permanent approximation`

## 교훈 / 다음에 적용할 점

1. **UI/시각 효과 검증 워크플로에서 headless 결과를 단독 신뢰하지 말 것** — CRITICAL #3 "브라우저 3단계 검증"을 실 브라우저에서 최소 1회 수동 수행. 특히 3D/WebGPU/카메라 조작 경로.
2. **"컴파일 성공" / "비검정 canvas" 같은 coarse assertion은 부분 성공을 탐지 못함**. browser-verify 스크립트에 도메인 특화 검증(특정 색상 픽셀 존재, scene object visibility)을 추가하되, 이것도 swiftshader freeze 상황에서는 false positive 가능.
3. **PM 계약에 "M1 백업 경로" (실패 시 대체안)를 명시적으로 박제**하면 sub-agent가 실패 판정 후 자동 전환 가능. 재승인 없이 진행.
4. **교차검증(Gemini)으로 발견한 옵션이 부분 성공할 수 있음** — 완전 실패 아닌 partial 자산으로 보존하고 ADR에 "향후 디버깅 자산" 명시. Frustum Corner Interpolation 코드는 lensing 효과 자체는 성공했으므로 `?ray3d=1` 옵트인 경로로 유지.
5. **QA 판정 기준에 "실 Chrome 수동 검증" 항목 표준화** — `status:review` 전이 전 체크리스트에 명시.

## 관련 노트/링크

- ADR: https://github.com/coseo12/astro-simulator/blob/main/docs/decisions/20260418-p7-track-b-ray3d.md
- P7-C PR: https://github.com/coseo12/astro-simulator/pull/217
- P6-B 선행 ADR (D' 변형 박제): https://github.com/coseo12/astro-simulator/blob/main/docs/decisions/20260417-accretion-disk-shadow-pipeline.md
- Iyer-Petters 2007, General Relativity & Gravitation (impact parameter b 관측자 좌표계 투영)
