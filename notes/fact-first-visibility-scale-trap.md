---
title: Fact-First 순수주의가 가시성을 침식하는 구조적 실패 — 사실 비율 원칙 + 동적 스케일 부재의 결합 함정
type: knowledge
source_repo: coseo12/astro-simulator
source_issue: 68
captured_at: 2026-04-23
status: refined
tags: [fact-first, visualization, scale, ux, design-principle, scientific-rendering]
related:
  - ./swiftshader-headless-pixel-freeze.md
  - ./hidden-constant-proportion-drift.md
---

## 요약

"사실(fact)은 1차, 시각은 2차 overlay" 같은 엄격 원칙 + **동적 스케일 부재** (고정 상수 배수 `×1` / `×500` 등) 의 결합은 **구조적으로 실사용에서 가시성이 깨지는 UX** 를 생성한다. 선언 원칙 레벨에서는 논리적으로 무결해 보이지만, 실 디스플레이에서 body 크기 차이가 극단적 (예: 태양:지구 = 109:1 → 태양 64px 일 때 지구 0.59px = sub-pixel) 이면 "focus 해도 대상이 안 보이는" 실사용 실패를 피할 수 없다. **완료된 Phase 의 성공 기준이 실사용에서 깨지는 현상** 의 전형.

## 본문

#### 발견 맥락

astro-simulator 프로젝트는 P10 에서 `docs/principles/fact-first.md` 박제 후 `educational`(과장 배수 ×20/×500) / `scientific`(1:1 실측) 이중 모드 도입. scientific 모드의 float32 jitter 를 해소하기 위해 P11-A Floating Origin (#288 / PR #291) 까지 구현·머지 완료. 그러나 **DoD 4 실 Chrome 육안 검증** 중 사용자가 직접 관찰:

> "focus 해도 행성이 안 보여 그리고 궤도에서 움직임도 보여야하는데 근본적으로 스케일에 대한 토론이 필요해"

자동화된 browser-verify (canvas pixel readback / console error 체크) 는 전부 PASS 였다. 실사용 실패는 **자동 검증이 포착할 수 없는 종류** — 렌더 파이프라인은 정확히 동작하지만 "화면에 의미 있는 무언가가 보인다" 는 조건이 DoD 에 들어있지 않았다.

#### 구조적 원인 분석

| 축 | 원칙 | 결과 |
|---|---|---|
| 사실성 | 상대 비율 실측 고정 | body 간 비율은 IAU 2015 기준 정확 |
| 시각성 | 동적 스케일 부재 | 고정 ×1 (scientific) → sub-pixel / 고정 ×20/×500 (educational) → 임의 비율 왜곡 |

두 축 중 **어느 하나를 지키려면 다른 하나가 깨지는 구조**. Educational 은 "과장 배수 ×20/×500" 이 임의 상수 (뷰포트/해상도 무관 — 왜곡) 라 사실성 위배. Scientific 은 비율 보존 이득이 sub-pixel 에 의해 **사실상 관찰 불가** 로 가치 소실.

#### 해결 원칙 (재설계 합의)

사용자가 제시한 5원칙:
1. 상대 비율은 실측 고정 (왜곡 금지)
2. **절대 스케일은 디스플레이 함수** — 뷰포트/zoom 에 따른 공통 동적 배수
3. 모드 통일 (이중 모드 분리 자체 폐기)
4. 거리도 동일 스케일 (궤도 반지름 포함)
5. 화면 이동은 자연스러워야 함

핵심은 **"뷰포트의 함수로 스케일을 정의"** 하는 관점 전환. 고정 상수 대신 tier 기반 동적 renderScale. 모든 body 는 동일 배수로 확대되므로 실측 비율이 자동 보존되면서 가시성도 확보.

#### 체크리스트 — 유사 프로젝트에서 사전 감지

과학 시각화 / AR/VR / 교육 / GIS / 분자 시뮬 / 천체 시뮬 등 **극단적 스케일 차이** 가 있는 제품 초기 설계 시:

- [ ] "실측 비율을 그대로 렌더링" 요구가 **뷰포트 해상도** 와 함께 명시되는가? ("1280×800 에서 N px" 류)
- [ ] "최소 가시 크기" (min-visible-px floor) 혹은 **뷰포트 기반 동적 스케일** 전략이 설계에 있는가?
- [ ] 이중 모드 분리 (sim/edu) 가 아닌, **단일 동적 모드** 로 통일 가능한지 검토했는가?
- [ ] DoD 에 **"화면에 의미 있는 무언가가 보인다"** 항목이 수치로 포함됐는가? (예: focus 대상 ≥ N px)
- [ ] 브라우저 자동 검증이 "canvas 가 검정이 아님" 이상의 **도메인 특화 pixel 검증** 을 포함하는가?

#### 재사용 가능 원칙

> **엄격 원칙 + 동적 적응 부재 = 실사용 실패.** 
> 원칙은 "사실/정확/무결" 같은 단일 축에서 강하게 선언되기 쉬우나, 실제 UX 는 뷰포트·해상도·카메라 거리 같은 **동적 문맥** 과 상호작용한다. 원칙 박제 직후 "이 원칙이 실 뷰포트/디바이스 분포에서 어떻게 작동하는가" 를 시뮬레이션해야 한다.

> **완료된 Phase 의 성공 기준이 실사용에서 깨지는 현상** 은 자동 검증으로 감지 불가. **사용자 실 디바이스 육안 검증** 을 DoD 의 일급 시민으로 포함.

## 관련 노트/링크

- astro-simulator #288 (P11-A Floating Origin), #298 (P12 Scale Unification 재설계 계약)
- ADR `docs/decisions/20260423-display-relative-scale-unification.md`
- docs/principles/fact-first.md (P10 원칙 박제 문서)
- volt #33 (headless 브라우저 false positive — 본 사례의 형제 교훈. headless PASS 여도 실 Chrome 에서 다를 수 있다)
- CLAUDE.md "빌드 성공 ≠ 동작하는 앱" 교훈의 원칙·UX 버전 확장
