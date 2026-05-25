---
title: R4 4 라운드 amendment 회고 — 사실 비율 vs 사용자 시각 인지 mismatch (함정 #7 신규)
type: report
source_repo: coseo12/astro-simulator
source_issue: 118
captured_at: 2026-05-25
status: refined
tags: [r-phase, dod-pass-not-product, measurement-only, lod, visual-fidelity, amendment-rounds, forensic-adr, sub-agent-orchestration]
related:
  - ./fact-mode-unification-ux-regression.md
  - ./reviewer-qa-gate-skip-headless-self-pass.md
---

## 배경/상황

태양계 시뮬레이터 R4 (지구 + 달 시각화) Phase 진입에서 **본 진입 + 4 라운드 amendment** 가 누적된 회고. 각 라운드는 sub-agent (PM / architect / developer / QA) 단계를 거치고 cross-validate (Gemini → Antigravity agy) 합의를 받았음에도, **모든 시각 회귀를 D-T2 사용자만 발견**.

## 시계열 (사례 5건 + 회귀 5건)

| 단계 | 결과 | 자동 검증 | D-T2 발견 회귀 |
|------|------|----------|----------------|
| R4 본 (#532, PR #537) | 머지 | r1-guard PASS / QA headless PASS / DoD 11/11 PASS | **moon mesh fusion** — earth disc 안쪽 박힘 (mesh radius 1.39e9 m > earth-moon distance 3.847e8 m 3.6배) |
| Amendment 1 (#532) | D8 임계 완화 | perspective 보정식 확정 (16.40% vs 식 예측 14.67% +11.8%) | 식 결함 자체는 자동 발견 가능 (식 vs 실측 mismatch 패턴) |
| Amendment 2 forensic (#539, PR #540 ADR + PR #542 fix close) | 박제 | cross-validate Gemini 합의 (5조건 5/5) | **moon focus LOD 함정** — visual scale=30 + moonScale=800 결합으로 카메라가 mesh 안쪽에 박힘 (cameraRadius/moonScaling = 1.011) |
| Amendment 3 (#539, PR #543 ADR + PR #545 fix) | 박제 | cross-validate Gemini 합의 + QA D3.1~D3.6 PASS (실 Chrome HEAD=1) | **사실 비율 vs 시각 인지 mismatch** — moon = earth 27.2% 사실 비율이 사용자 천문 직관에서 "비정상적으로 큼" |
| Amendment 4 (#539, PR #545 추가 commit) | 박제 | (architect 재호출 생략, 메인 직접 변경) | LOD billboard sub-pixel culling — 후속 #546 분리 |

## 내용

#### 7 함정 패턴 (R4 누적 발견)

1. **r1-guard `boundingSphere.radiusWorld` 측정 한계** — mesh 가 *존재* 하는 위치만 측정, 다른 mesh 와의 시각적 fusion 검출 못 함
2. **diskArea brightness threshold sub-pixel 함정** — moon disc area 0.01% 는 sub-pixel level 인데 회귀 임계 (baseline 대비 변동) 만 검증, 절대 인지 임계 없음
3. **Headless 측정 시각 인지 mismatch** (volt #74 + #78 변형) — Playwright headless (swiftshader) 의 LOD low override 가 사용자 인지와 다른 결과
4. **DoD "renderable" 만 정의, "visually distinguishable" 미정의** — boundingSphere ≥ Npx 만 검증, mesh 간 시각 분리 명시 없음 (3 라운드 정확 재발)
5. **Architect ADR 수학적 가능성 사전 검증 누락** — moon mesh radius vs earth-moon distance 산출 후보 비교에 부재 (3.6배 / 13.26배 모순을 후속 debug 로 발견)
6. **Cross-validate Gemini blind spot** — 단일 모델 + 다른 모델 둘 다 같은 수학적 모순을 놓침 (sanity check 가 cross-validate 의무에 없음)
7. **(신규) 사실 비율 vs 사용자 시각 인지 mismatch** — moon = earth 27.2% (radius 사실 비율 정합) 가 사용자 천문 직관 (moon = sub-pixel) 과 충돌. 사실 비율 정합 자체가 "올바른 결정" 으로 5 단계 합의됐지만 D-T2 만 발견

#### Measurement-only DoD 함정 정확 재발 (함정 #4)

Amendment 2 fix (PR #542) QA 단계 — D2.4 "moon focus 화면 중앙 ≥ 200 px disc" 측정-only 임계. developer 가 196.65 px 측정값 박제했으나 **`isVisible=false` 인 high variant 의 boundingSphere projection 결과** — 실제 화면에 안 그려짐. QA 가 실 Chrome HEAD=1 으로 발견. Amendment 3 가 **LOD-aware measurement 패턴** (`isVisible === true` AND `pxDiameter ≥ N` 둘 다 만족) 을 SSoT 박제로 해결.

#### LOD × visual scale 결합 결함 (Amendment 3 의 의미)

orbit visual scale=30 + moonScale=800 결합:
- moon high variant wsRadius ≈ 60426 (visual scale 적용)
- focus 진입 식 `cameraRadius = meshRadius × 5` → 302130 (mesh 표면 안쪽 0.01)
- screenCoverage ≈ 0 → LOD `low` 판정 → high `isVisible=false`
- moon-lod-low 1 px billboard 만 가시
- **visual scale 30 → 50/75 fallback 으로 해결 불가** (LOD 전환 자체가 작동 안 함)

해결: `FOCUS_USER_RADIUS_MULTIPLIER_SATELLITE = 20` 신규 SSoT + camera focusOn 식 satellite 분기. tier 전환 race (applyFocusTier 직후 mesh.scaling 점프) 도 2차 측정 패턴으로 차단.

## 교훈 / 다음에 적용할 점

#### 1. R-Phase ADR 의무 가드 추가 (3건)

- **수학적 양립 가능성 표** — visual scale × mesh radius × distance 매트릭스 의무 (mesh 가 distance 흡수하는지 사전 산출). architect 단계 의무 박제
- **LOD-aware measurement 패턴** — 모든 px 측정 DoD 에 `mesh.isVisible === true` AND `pxDiameter ≥ N` 의무. Concrete Prediction §수치 DoD 의무 인용
- **시각 분리 가능성 (visually distinguishable) DoD** — body 간 pixel distance > sum mesh pixel radius 검증 + 사용자 인지 단위 (천문 직관) 정합 점검

#### 2. Cross-validate prompt 보강

- "수학적 양립 가능성 sanity check" 명시 질문 추가 — Gemini 가 자동 발견 못 한 함정 유형
- "사용자 인지 단위 (천문 직관 / mental model) 와 사실 비율 박제값 충돌 점검" 명시 질문 추가 (함정 #7 회피)

#### 3. QA scenario 보강

- 시나리오 1 (정적) 에 "각 body 가 별도 disc/dot 로 분리 보이는가" 시각 분리 검증 명시
- 시나리오 2/3 (focus/URL) 에 "사용자 인지 단위에서 자연스러운 크기인가" 시각 정합 검증
- **headless 측정의 false negative 회피 — 실 Chrome HEAD=1 의무**

#### 4. 사실 비율 박제값 함정 (신규 함정 #7) 회피 패턴

- "사실 비율 정합" 이 항상 올바른 결정이 아님 — **사용자 천문 직관** (moon = sub-pixel) 과 충돌 시 사용자 인지 우선
- R-Phase ADR §결정 비교 표에 "사실 비율 정합 vs 사용자 시각 인지" 직교 축 추가 의무
- moonScale 사례: 사실 비율 (800, earth 27.2%) → 인지 정합 (200, earth 6.8%) 변경

#### 5. Sub-agent 오케스트레이션 — 매 라운드 D-T2 가 유일 발견자 패턴

- 5 단계 (PM / architect / cross-validate Gemini / developer / QA) 모두 합의했지만 사용자만 발견하는 회귀는 **자동 검증 인프라의 본질적 blind spot**
- D-T2 사용자 검증을 "마지막 통과 단계" 가 아닌 "유일한 진실 단계" 로 인식
- CRITICAL #3 의 "UI 작업 빌드 후 3단계 검증" 을 sub-agent 가 자체 검증해도 D-T2 의무는 사용자 의무로 분리 (volt #77 의 강화)

#### 6. Amendment 라운드 누적 패턴 = forensic ADR 변형 유지

- R4 ADR 가 Amendment 1 → 2 (forensic 승격) → 3 → 4 누적
- forensic 변형 의 8 섹션 구조가 amendment 라운드 N 의 cross-link + 의사결정 trace 유지 효과적
- R5+ R-Phase ADR 진입 시 "forensic 변형 발동 조건 사전 자기 점검" 의무 (3 라운드 이상 예상 시 사전 승격)

## 관련 노트/링크

- harness volt #72 / #74 — 사실 모드 단일화 (DoD PASS ≠ UX 가시성) — 본 사례의 1차 박제
- harness volt #77 — developer 후 reviewer/qa 건너뛰기 함정
- astro-simulator #532 (R4 본) / #539 (forensic followup) / #546 (LOD UX 후속)
- PR #537 (R4 본 머지) / #540 (Amendment 2 ADR) / #543 (Amendment 3 ADR) / #545 (Amendment 3+4 fix 통합) / #542 (Amendment 2 fix close)
- ADR 박제: docs/decisions/20260520-r4-earth-moon-visualization.md (forensic 변형 + Amendment 1~4)
- CLAUDE.md §"DoD PASS ≠ 제품 동작" (volt #74) + §"headless 브라우저 검증 ≠ 실 브라우저" (volt #78)
