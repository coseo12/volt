---
title: 은닉 상수 (dead reference) 의 상대 비율 drift — 주석 계약 vs 구현 drift 교훈의 숨은 상수 변형
type: knowledge
source_repo: coseo12/astro-simulator
source_issue: 69
captured_at: 2026-04-23
status: refined
tags: [reviewer, static-review, drift, hidden-constant, proportion-bug, adr-prediction, code-review-pattern]
related:
  - ./comment-contract-implementation-drift.md
  - ./new-data-not-new-code-adr-prediction.md
  - ./fact-first-visibility-scale-trap.md
---

## 요약

모듈 A 에서 상수를 폐기하고 동적 함수로 교체해도, **다른 모듈 B/C/D 에 같은 이름의 상수가 독립 선언된 채 잔존** 하면 전체 시스템이 상대 비율 drift 버그를 조용히 생성한다. 단일 모듈 grep 은 이를 놓친다 — **"의미 레벨 검증"** (단위 테스트로 상대 비율 불변식 검증) 이 필요하다. CLAUDE.md 기존 "주석 계약 vs 구현 drift" 교훈의 **숨은 상수 변형** — ADR Concrete Prediction 이 이를 실시간 감지 가드 역할.

## 본문

#### 발견 맥락

astro-simulator P12-A (#298 / PR #301) 재설계 작업에서 `SCENE_UNIT_PER_METER = 1/AU` 하드코딩을 **tier 기반 동적 함수** (`renderScaleForTier(tier)`) 로 교체. 주 모듈 (`solar-system-scene.ts`) 에서는 제거 완료. developer 는 "R4 DoD (SCENE_UNIT_PER_METER 동적화) PASS" 선언.

Reviewer 정적 리뷰에서 **3 개 위성 모듈** 에 독립 상수 선언 발견:

```
packages/core/src/scene/asteroid-belt.ts:23     const SCENE_UNIT_PER_METER = 1 / AU;
packages/core/src/scene/ring-placeholder.ts:26  const SCENE_UNIT_PER_METER = 1 / AU;
packages/core/src/scene/ring-shader.ts:41       const SCENE_UNIT_PER_METER = 1 / AU;
```

각 파일이 **자체 상수 선언 + 자체 사용**. 주 모듈 grep 만으론 보이지 않는다.

#### 버그 영향 (상대 비율 drift)

T2/T3 tier 에서:
- 행성 mesh = `renderScaleForTier(tier)` 로 확대 ✓
- Rings / asteroid belt = **AU 고정 상수 유지** ✗

결과: 행성은 커졌는데 고리/소행성대는 그대로 → **body 간 실측 비율 drift**. 사용자 원칙 #1 (실측 비율 고정) 과 #4 (거리 동일 스케일) 정면 위배. 사용자는 렌더 결과만 보면 "고리가 점점 작아 보인다" 만 체감, 원인 불명.

#### ADR Concrete Prediction 의 실시간 감지 가드

이 버그는 **ADR `Concrete Prediction #2`** 와 직접 연결:

> "tier 경계 조정 시 렌더 경로 코드 변경 0 줄"

PR #301 에서 tier 를 조정하려 해도 rings / asteroid 모듈은 **수정 필요 상태** (tier 인자 받는 인터페이스로 재설계). 즉 본 PR 단계에서 이미 예측 실패. Concrete Prediction 이 "추상화 건강성 지표" 로 작동하여 drift 감지 역할.

#### 해소 전략

CLAUDE.md 교훈 확장 — "숨은 상수 drift":

1. **파괴적 상수 제거 PR 은 반드시 "의미 레벨" 테스트 가드** — 단위 테스트로 "모든 body 간 scale 비율이 실측 비율과 일치" 불변식 검증 (`tier-proportion.test.ts` 5건)
2. **상수 이름 키워드로 전체 저장소 grep** — 주 모듈만 보지 말고 `grep -rn "<CONSTANT_NAME>" <lang dirs>` (단, 주석·테스트 제외)
3. **ADR Concrete Prediction 박제** — "이 추상화로 미래 변경 시 X 코드 0줄" 예측을 PR 단계에서 실측으로 재현
4. **reviewer sub-agent 의 grep 독립 수행** — developer 가 놓친 상수/참조를 별도 세션에서 재검색

#### 패턴: Dead Reference 위험

본 사례의 자매 케이스로 `scale-badge.tsx` 의 `MAX_SCALE_BY_KIND` 인라인 미러링 + 주석 `visual-scale.ts 와 SSoT 동기` 가 **dead reference** (참조 대상 파일이 폐기됨). UI 가 여전히 "×N 과장 중" 표시 → 사용자에게 거짓 정보.

> **"SSoT 동기" 주석은 SSoT 파일이 폐기되면 거짓 메타데이터 로 전락**. 파괴적 리팩토링 PR 은 주석에 박제된 SSoT 참조도 함께 재검토.

#### 체크리스트 — reviewer / developer 정적 리뷰 시

- [ ] 주 모듈에서 상수 제거 시 **저장소 전체 grep** 으로 독립 선언 잔존 확인 (`grep -rn "<CONST_NAME>" src/`)
- [ ] 주 모듈에서 export 제거 시 **주석에 박제된 SSoT 참조** 도 함께 검증 (dead reference)
- [ ] 단위 테스트에 "상대 비율 불변식" 포함 (모든 모듈 공통 배수로 확대되는지)
- [ ] ADR Concrete Prediction 대비 실측 diff (예: `git diff --stat <추상화 경로>` 로 예측 성공 재현)
- [ ] UI 표시 문구가 삭제된 기능을 여전히 언급하는지 (dead UX reference)

## 관련 노트/링크

- astro-simulator PR #301 (reviewer 1차 request_changes 에서 B1/B2 발견 → 수정 → Round 2 approved)
- astro-simulator ADR `docs/decisions/20260423-display-relative-scale-unification.md` § Concrete Prediction
- CLAUDE.md "주석 계약 vs 구현 drift — 버그 생성원" 교훈 (volt #49)
- CLAUDE.md "신규 데이터 ≠ 신규 코드 — ADR 예측 재현" (volt #47)
- volt #49 주석-구현 drift 의 숨은 상수 변형
