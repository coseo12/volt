---
title: 원칙 선언 직후 cross-validate — Claude 편향 4종 + ADR 재도입 트리거
type: knowledge
source_repo: coseo12/astro-simulator
source_issue: 55
captured_at: 2026-04-20
status: refined
tags: [cross-validate, sprint-contract, adr-framing, bias-correction, principle-declaration]
related:
  - ../meta/patterns/cross-validate-after-policy-freeze.md
  - ./cross-validate-accept-vs-defer.md
  - ./cross-validate-self-application-positive-path.md
  - ./external-tool-claim-empirical-verify.md
  - ./ci-multi-language-design-4-principles.md
---

## 요약

프로젝트 상위 원칙을 선언한 직후 교차검증(Gemini)을 적용한 첫 프로젝트 레포 사례 (astro-simulator P10 Phase 설계, 2026-04-20). volt #23/#42 의 "박제 직후 1회" 루틴이 ADR·정책 수준을 넘어 **원칙 선언 수준**에서도 작동함을 확인. Claude 편향 4종을 실측 + ADR 에 "재도입 트리거 섹션" 필수화 라는 재사용 가능한 패턴 2건 도출.

## 본문

### 1. 적용 맥락의 확장

volt #23 (정책/설계 박제 직후 교차검증), #42 (4회 연속 성공) 는 모두 harness-setting 레포 **내부** 관찰이었음. 본 사례는 **프로젝트 레포** (astro-simulator, Babylon.js WebGPU 천체 시뮬레이터) 에서 **상위 원칙 선언** (Fact-First, Visual-Second) 직후 적용한 첫 유의미한 사례.

원칙 선언은 ADR 보다 추상도 높고 코드 계약보다 상위. 이 수준에서도 cross-validate 루틴이 강력한 발견을 끌어내는지 검증 대상이었음.

### 2. Gemini 가 포착한 고유 발견 6종 (재사용 가능)

Claude 설계에 없었던 지적들:

1. **암묵 전제의 빈틈** — "사실"의 범주가 공간(radius/orbit)만 전제했으나 **시간(J2000.0 Epoch)** 도 동등한 사실 축이라는 지적
2. **원칙의 실용적 실패 예측** — "사실 모드 1.0 스케일 → 행성 sub-pixel → 사용자 이탈" 시나리오를 **원칙 적용 전에** 감지
3. **불확실성의 원칙 포함성** — "IAU 공식값이 없는 것도 사실 (우리가 얼마나 모르는가)". Fact-First 가 '완벽한 값'이 아닌 '오차까지 포함한 지식 상태' 여야 한다는 재정의
4. **비기능 요구 누락** — 저작권/라이선스 crediting (IAU/NASA attribution) 외부 계약 조건
5. **회귀 측정 누락** — 물리 수정 후 벤치 재측정 단계 부재 → 신규 sub-phase (P10-D.5) 도출
6. **ADR 재도입 트리거 섹션** — 폐기/보류 결정에서 "다시 고려할 조건" 명시 필수 (아래 재사용 패턴 B 로 승격)

### 3. Claude 편향 4종 — 이견 수용 기반 분류

교차검증에서 Gemini 이견이 합리적으로 판명되어 Claude 가 수정 수용한 4개 사례. 향후 같은 편향을 사전 감지할 수 있는 체크리스트:

| # | 편향 | 본 사례 | 보정 방향 |
|---|---|---|---|
| 1 | **낙관적 일정 산정** | P10-B 데이터 감사 3d | body 50+ 전수 IAU 대조는 리서치 시간 가중 → 3~5d |
| 2 | **결합 관계 간과** | P10-C (UI) / P10-D (물리) 병렬 가능 주장 | 과장 모드와 물리 수정이 동시 들어가면 회귀 원인 추적 불가 → 직렬 |
| 3 | **폐기 프레이밍 선호** | "모바일 영구 폐기" | 기술 성숙도는 시간 함수 → "무기한 중단 + 재도전 조건" 프레이밍 |
| 4 | **순수주의 원칙 적용** | "사실 모드를 디폴트" | 디폴트는 교육 관례 (과장) 유지 + 1-클릭 사실 모드 보장 (아래 재사용 패턴 A) |

### 4. 재사용 패턴 A — 원칙의 실질적 구현 = "접근성 보장"

원칙 "사실 1차"를 문구 그대로 구현 (디폴트=사실모드) 하면 UX 실패. 문구를 수정하지 않고 **운영 해석** 으로 흡수:

> "디폴트는 교육용 관례 (과장 + 명시 배지 + 온보딩) 를 유지하되, 사실 모드가 **항상 1-클릭/1-URL 거리** 에 있어야 한다"

**일반화**: 원칙의 실질적 구현 = "기본값의 엄격함" 이 아니라 "대안 경로의 접근성 보장" 인 경우가 있음. 원칙을 문구 그대로 적용하면 사용자 첫 인상을 망칠 때, 문구를 바꾸지 말고 "언제든 원칙 경로에 접근 가능" 으로 재해석.

### 5. 재사용 패턴 B — ADR "재도입 트리거 섹션" 필수화

기술 선택 폐기/중단 ADR 에 **재도입 트리거 섹션** 필수 포함.

**본 사례** (모바일 지원 보류 ADR):
- 재도전 트리거: "WebGPU 모바일 호환성 안정화" / "LOD 도입 후 iPad Pro M2 에서 60fps 달성"
- tier 전략: tier-a (desktop discrete + iPad Pro M) / tier-b (integrated + Android flagship) / tier-c (entry-level Graceful Degradation)
- Graceful Degradation UX: 모바일 접속 시 안내 모달

**일반화**: 재도입 트리거가 없으면 결정이 "죽은 코드/기능의 영구 부채" 로 고정. 시간 함수인 기술 성숙도를 결정 시점에 동결하는 것은 오류.

**ADR 섹션 템플릿 갱신 제안**:
- 배경 / 후보 비교 / 결정 / 결과·재검토 조건 / **재도입 트리거** / **Graceful Degradation UX**

### 6. 교차검증 결과 자체의 자산화 루프

본 세션에서 교차검증 보고서가 그대로 스프린트 계약 문서 (`p10-plan.md`) 의 "교차검증 반영 사항" 섹션으로 편입됨. PR 본문에도 재인용. 교차검증이 일회성 검토가 아니라 **계약 문서에 박제되는 지식** 으로 승격되는 흐름 확립.

**운영 규칙 제안**: 교차검증 결과의 (a) 합의 / (b) 이견 수용 / (c) 고유 발견 분류를 스프린트 계약 문서에 항상 포함하는 섹션으로 고정.

### 7. 선행 volt 이슈와의 차별점

- #23 (정책 박제 직후 효율) / #42 (4회 연속 성공): **관찰 횟수** 축적
- #29 (수용 vs 후속 분리 3단 프로토콜): **판단 규칙** 정립
- #51 (외부 툴 동작 주장 오탐): **Gemini 한계** 감지
- **본 이슈**: **적용 범위 확장 (원칙 수준)** + **Claude 편향 체크리스트** + **ADR 템플릿 개선 제안**

## 관련 노트/링크

- astro-simulator PR #266 (P10 플래닝) — https://github.com/coseo12/astro-simulator/pull/266
- 선행 volt #23 / #29 / #42 / #51
- CLAUDE.md "cross-validate" 섹션 (하네스 공통 루틴)
- astro-simulator v0.9.0 (P9 완료) 직후 방향 재조정 세션 산출물
