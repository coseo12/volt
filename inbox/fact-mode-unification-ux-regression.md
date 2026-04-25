---
title: 사실 모드 단일화 부작용 — 모든 DoD PASS 에도 UX 가시성 회귀
type: knowledge
source_repo: coseo12/astro-simulator
source_issue: 72
merged_captures: [74]
captured_at: 2026-04-25
status: inbox
tags: [ux-regression, adr-principle, fact-first, dod-gap, phase-handoff, billboard-marker, multi-phase-merge, rendering, ui-visibility]
related:
  - ../notes/fact-first-visibility-scale-trap.md
  - ../notes/swiftshader-headless-pixel-freeze.md
  - ../notes/lint-staged-gitignore-silent-revert.md
  - ../notes/roi-test-skip-judgment-drift.md
---

## 요약

P12 Display-Relative Scale Unification ADR (`20260423-display-relative-scale-unification.md`) 이 `educational` / `scientific` 모드 토글을 폐기하고 단일 사실 모드로 전환했다. Phase 분리 (A/B/C) 까지 모든 DoD PASS 로 머지됐고 P11-B (LOD) + P11-C (GPU tier) 까지 누적 머지됐으나, 사용자가 브라우저에서 확인한 기본 화면은 **궤도 라인 + 해왕성 1개만 보이는 빈 화면**. 태양조차 sub-pixel. 사용자 표현: "초기 기획 상태보다 못한 상태". 수치 DoD 전부 PASS 였음에도 UX 회귀. `빌드 성공 ≠ 동작하는 앱` / `HTTP 200 ≠ 올바른 리소스` 의 **UX DoD 버전**. 원칙 폐기 ADR 은 downstream UX 계약 재검증을 동반해야 한다.

> 본 노트는 동일 사건에 대한 두 차례 캡처(volt#72, volt#74)를 통합한 결과다. 첫 캡처(#72)는 메타 분석·후속 결정에 비중을 두었고, 보강 캡처(#74)는 콘솔 에러 정보·"초기 진입 상태 스크린샷" 교훈을 추가했다. 두 캡처의 차이는 본문 안에 모두 반영했다.

## 본문

#### 현상

- P12 ADR 은 `educational` / `scientific` 토글 폐기 → 단일 사실 모드
- Phase 분리 (A/B/C) 완료, 모든 DoD PASS:
  - Phase A: Scale Tier 엔진 + 3단 tier (Solar/Inner/Body) + 하이브리드 트리거
  - Phase B: 8D 카메라 dolly + apparent size 불변
  - Phase C: UI 4건 제거 + educational 모드 완전 폐기
- browser-verify 정적/인터랙션/흐름 3단계 자동 검증 모두 통과
- 이후 P11-B (LOD) + P11-C (GPU tier) 까지 누적 머지
- **사용자 수동 브라우저 확인** 결과: 기본 진입 화면이 궤도 라인 + 해왕성 1개 (우측 35 AU) + 중앙 흰 점 (태양인지 불명). 수성/금성/지구/화성/목성/토성/천왕성 전부 invisible
- URL `?focus=earth` 등이 전혀 반영 안 됨 (실측 playwright 확인)
- 상단 shortcut 버튼은 태양/지구/목성/해왕성 4개뿐 (수성/금성/화성/토성/천왕성 직접 접근 수단 부재)
- 콘솔 에러는 favicon 404 + integrated GPU 폴백 warning 정도로 실제 렌더에 영향 없음

#### 근본 원인 4중 gap

1. **Store `focusBodyId` 필드 부재**
   - `apps/web/src/store/sim-store.ts` 의 state 에 `focusBodyId` 없음, `selectedBodyId` 만 존재
   - URL `?focus=earth` 등이 store 에 반영될 필드 자체 부재 — playwright 실측 확인
   - `packages/core/src/scene/tier.ts` 는 `focusBodyId` 를 인자로 사용 → scene 과 store 미통합
   - URL `?focus=X` parser 가 저장할 곳이 없음

2. **Solar tier 기본 화면 = 빈 궤도**
   - 태양계 실측 스케일: 태양 1.4M km vs 거리 1.5억 km → Solar tier 에서 태양도 sub-pixel
   - 모든 내행성 + 외행성 sub-pixel 로 invisible
   - Fact-First 원칙 선언 1조 "디폴트 UX 는 교육용 관례(`educational` 모드) 유지" → P12 에서 **폐기**
   - Fact-First 선언 "사용자는 1-클릭/1-URL 로 사실 모드 접근 가능" 이 **반대 방향** (사실 모드가 디폴트) 으로 구현됨
   - Fact-First 1조 폐기 후 **대체 UX 계약 부재**

3. **Shortcut 4개만**
   - 상단 버튼: 태양 / 지구 / 목성 / 해왕성 / reset
   - 수성 / 금성 / 화성 / 토성 / 천왕성 직접 접근 수단 없음
   - 사용자 보고 "연구 모드에서 포커스 시 일부만 보임" = 이 4개만 클릭 가능

4. **P11-B billboard marker 통합 미완결**
   - `roadmap-v2-solar-precision.md` P11 노트: "billboard marker 통합 (P11-B) 후속 ongoing"
   - sub-pixel body 의 가시성 marker 가 동작하지 않음
   - 후속 라벨이 **blocker 로 승격되지 않음** — 다음 Phase 진행의 차단 요건이 아니었음

#### 왜 모든 DoD PASS 였음에도 회귀였는가 (메타 분석)

- DoD 수치 (`screenshot diff < 15%`, `bench 회귀 < 5%`, `idle fps ≥ 30`) 가 전부 **성능·정합성 지표** — UX 회귀 감지 축이 아님
- "브라우저 3단계 검증" 도 **지구 focus 상태 위주** 수행. **초기 진입 상태 = 빈 화면** 이 검증 범위 밖. QA 가 `?focus=earth` 스크린샷은 찍었지만 default URL 스크린샷은 찍지 않음
- P12 ADR 의 `## 결과·재검토 조건` 에 **UX 계약이 빠져있음** — Concrete Prediction 3건 모두 "코드 변경 0 줄" 같은 구조 예측이고, "사용자가 기본 화면에서 태양계를 탐색 가능" 같은 UX 예측 없음
- 로드맵 P11 노트 "billboard marker 통합 후속 ongoing" 이 P12/P11-C 진행의 **블로커로 간주되지 않음** — 후속 라벨이 행동 변화 없이 라벨만 남음

#### 일반화된 교훈

1. **"완료된 DoD" 와 "사용자가 인지하는 제품" 이 분리 가능**
   - 수치 DoD 전부 PASS 여도 UX 회귀 가능
   - `빌드 성공 ≠ 동작하는 앱` / `HTTP 200 ≠ 올바른 리소스` 의 **UX DoD 버전**
   - "스크린샷 캡처는 Level 1 에 불과. '렌더링 됨 = 동작함' 이 아니다" (CLAUDE.md 실전 교훈) 의 UX 회귀 확장

2. **원칙 폐기는 원칙 폐기다**
   - ADR 이 "educational/scientific 토글 폐기" 같은 상위 원칙을 수반할 때 downstream UX 계약 전체 재검증 **의무**
   - P12 ADR 은 Fact-First 원칙 1조 폐기를 수반했으나 재검증 blocker 로 표시되지 않음
   - 대안 루틴: 원칙 폐기 ADR 은 `## 영향받는 원칙` 섹션 + `## UX 회귀 체크리스트` 섹션 필수
   - 자동화: 원칙 폐기 ADR 은 자동으로 **영향 범위 grep + UX 회귀 검증 DoD** 를 편입

3. **"ongoing 후속" 라벨이 blocker 로 승격되는 조건 필요**
   - "billboard marker 통합 후속 ongoing" 같은 라벨이 **다음 Phase 블로커로 자동 승격** 되는 장치 부재
   - 대안: 라벨 `blocker-candidate:next-phase` + 다음 Phase PM 계약 시 필수 리뷰 트리거
   - 상위 원칙 영향 받는 ongoing 항목은 같은 원칙 수정 ADR 의 자동 blocker 로 간주

4. **UX DoD 를 성능 DoD 와 별도로 박제**
   - "기본 진입 화면에서 태양과 주요 행성 N개가 명시적으로 visible" 같은 **사용자 관찰 가능 동작** 이 DoD 에 포함되어야 함
   - 예: `DoD-UX-1: 기본 진입 후 3초 이내 화면에서 ≥5개 body 가 클릭 가능한 크기(≥4px)로 렌더된다`

5. **사용자 1차 수동 검증을 Phase 종료 DoD 에 포함**
   - browser-verify 자동화는 회귀 감지에 **불충분**
   - 특히 Fact-First/scale 같은 전역 UX 원칙 변경 시 실 사용자 수동 검증 의무
   - 대안: Phase 종료 DoD 에 "사용자 1차 수동 브라우저 탐색 후 피드백 포함" 명시

6. **초기 진입 상태 스크린샷을 QA 게이트에**
   - focus/interaction 상태만이 아니라 **URL 파라미터 없는 default 진입 상태** 가 주요 평가 대상
   - "첫 1초에 사용자가 뭘 보는가" 를 자동 캡처
   - `?focus=earth` 만 찍는 QA 패턴은 default 진입 회귀를 구조적으로 놓친다

#### 후속 조치 (사용자 결정, 2026-04-24)

- 기획 의도 폐기: 로드맵 v2 / ADR P10~P12 / Fact-First 원칙 전부 초기화
- 접근 전환: incremental body-by-body build ("태양계를 태양부터 하나씩 구성")
- 각 단계 DoD: **"사용자가 실제로 보이는 body"** 기준
- 로드맵 v2 / ADR P10~P12 / Fact-First 원칙이 결과적으로 UX 회귀 산출 → 폐기

## 관련 노트/링크

- astro-simulator 이슈/PR:
  - [#268](https://github.com/coseo12/astro-simulator/issues/268) P10-A Fact-First 박제
  - [#289](https://github.com/coseo12/astro-simulator/issues/289) P11-B LOD
  - [#298](https://github.com/coseo12/astro-simulator/issues/298) P12 Display-Relative Scale
  - [#310](https://github.com/coseo12/astro-simulator/issues/310) Tier 네이밍 정책
  - [#290](https://github.com/coseo12/astro-simulator/issues/290) P11-C
- CLAUDE.md 실전 교훈: "빌드 성공 ≠ 동작하는 앱", "HTTP 200 ≠ 올바른 리소스", "display-only 버그 패턴", "staging 성공 ≠ 커밋 내용" (volt #13)
- 관련 volt 이슈: #33 (headless 브라우저 검증 ≠ 실 브라우저), #13 (빌드 성공 ≠ 동작)
