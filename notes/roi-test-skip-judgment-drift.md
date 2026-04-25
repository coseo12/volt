---
title: ROI 5문 체크 대체 판정이 시각 회귀를 놓친 사례 — 순수 함수 추출 가능성 1%라도 있으면 추출 우선
type: knowledge
source_repo: coseo12/astro-simulator
source_issue: 71
captured_at: 2026-04-25
status: refined
tags: [testing, roi, pure-function, regression, contract, sprint-contract, dod, test-strategy]
related:
  - ./comment-contract-implementation-drift.md
  - ../meta/patterns/test-roi-rescope-pattern.md
  - ./fact-mode-unification-ux-regression.md
---

## 요약

CLAUDE.md 스프린트 계약 6항 "ROI 5문 체크" 로 단위 테스트 작성을 **대체 (주석 계약 + 인접 테스트 + bench 게이트)** 로 판정했으나 실제 시각 회귀가 발생한 사례. 회귀의 특성이 "조용히 퇴행" 이라 QA 브라우저 검증 없이는 감지 불가. **순수 함수 추출 가능성이 1%라도 있으면 주석 계약 대체보다 추출 + 단위 테스트를 기본 선택**. volt #49 "주석 계약 vs 구현 drift" 의 역방향 함정 — 구현 drift 가 아닌 **테스트 생략 판정의 drift** 이며, ROI 5문 체크 자체가 편향적 판정이 될 수 있음.

## 본문

#### 컨텍스트 (2026-04-23, astro-simulator #313 M2 PR #315)

렌더 경로 3지점에 `activeTier === 'body'` 조건부 분기를 추가하는 수정 (3줄 실질 변경). CLAUDE.md 스프린트 계약 6항 ROI 5문 체크를 수행:

| ROI 질문 | 판정 |
|---|---|
| 테스트 환경 구축 비용 > 5배? | ✅ Scene 인스턴스 + CameraController + FloatingOrigin 조합 mock 비용이 3줄 수정 대비 과다 |
| 몇 줄 보호? | ✅ 3줄 |
| 회귀 시 조용히 퇴행 vs 빌드 실패? | ⚠️ "조용히 퇴행" — M3 bench 재측정 게이트로 **자동 감지** 가능하다고 판단 |
| 인접 테스트로 간접 보증 가능? | ✅ `floating-origin.test.ts` 의 `reset()` / `setOriginToBody no-op` 테스트 |
| 미래 fixture 후 저렴? | ✅ Scene integration 테스트 인프라는 별도 이슈 후보 |

결론: "T3 기능 회귀 가드 단위 테스트 생략, 주석 계약 + 인접 property 테스트 + M3 bench 게이트로 대체". ADR §Amendments 박제.

#### 실제 결과: QA 브라우저 검증에서 시각 회귀 실측

| DoD | 허용 | develop baseline | PR | 판정 |
|---|---|---|---|---|
| V5 지구 세로 40% | 304~336px | 322px | **296px** | ❌ |
| A1 focus 중심 편차 | ≤10px | 0.0px | **119.9px** | ❌ |

3회 결정적 재현 (flaky 아님). 빌드/단위 테스트 328/328 pass, bench 게이트도 **focus 시나리오 개선** 으로 통과 가능한 범위. bench 게이트는 회귀를 **감지하지 못함** — bench 는 fps 만 측정하고 시각 정확성은 측정 대상 아님.

#### 왜 ROI 체크가 실패했는가 (근본 원인)

1. **"회귀 시 조용히 퇴행 vs 빌드 실패" 질문의 함정**: "조용히 퇴행" 을 "bench 게이트로 감지 가능" 으로 낙관 평가. 그러나 **bench 게이트는 성능 회귀만 감지하고 시각 정확성 회귀는 감지 대상 외**. 체크 질문 자체가 회귀 종류를 구분하지 않음
2. **"인접 테스트로 간접 보증" 의 범위 착각**: `floating-origin.test.ts` 는 FloatingOrigin 클래스 자체 계약만 보증. **setTier 의 tier 전환 로직** 은 검증 대상 외였음 — 다른 레이어의 테스트로 간접 보증한다는 판정이 잘못된 범위 설정
3. **순수 함수 추출 가능성 과소평가**: "Scene 인스턴스 mock 비용 과다" 로 판정했으나, **tier 분기 로직 자체는 Scene 과 무관한 순수 함수로 추출 가능** 했음 (`computeFloatingOriginForTier(tier, focusId, lookup)`). 수정 시점엔 "Scene 에 묶인 클로저" 라는 현 구조에만 주목하고 **추출 리팩터 여지** 를 고려하지 않음
4. **편향**: ROI 체크 자체가 "테스트 작성 비용을 과대 추정하고 대체 경로를 과소 추정" 하는 방향으로 편향. 스프린트 속도 압력 하에서 "5개 체크 모두 yes" 로 수렴하기 쉬움

#### 수정 방식 (번복)

`packages/core/src/scene/tier.ts` 에 순수 함수 추출:

```ts
export function computeFloatingOriginForTier(
  tier: Tier,
  focusBodyId: string | null,
  lookupFocusWorld: (id: string) => Vec3Double | undefined,
): Vec3Double | null {
  if (tier !== 'body') return [0, 0, 0];
  if (focusBodyId === null) return null;
  const focusWorld = lookupFocusWorld(focusBodyId);
  if (!focusWorld) return null;
  return [focusWorld[0], focusWorld[1], focusWorld[2]];
}
```

`setTier` 호출부는 반환값으로 `reset()` / `setOriginToBody()` 분기. 단위 테스트 8건 (T1/T2/T3 × focus 유/무/stale 6조합 + reference + `[0,0,0]` 계약) — Scene 인스턴스 없이 가능. vitest 234/234 pass. QA 재검증 V5 322px / A1 0.0px 복귀.

#### 교훈 (agent/skill 개선 포인트)

##### 원칙 (기본 선택)

**"순수 함수 추출 가능성이 1%라도 있으면 주석 계약 대체보다 추출 + 테스트를 기본 선택"**. 아래 조건 중 **하나**라도 해당하면 ROI 체크 없이 추출 우선:

- 분기 조건이 **입력 타입** (enum / discriminated union) 만으로 결정
- 사이드 이펙트가 **반환값 소비** 로 분리 가능 (`compute*` 패턴)
- 같은 함수를 **다른 컨텍스트에서 재사용** 할 여지 (e.g. tier 전환 외에 focus 전환 시에도 동일 로직 필요)

##### ROI 5문 체크 보강 권장

| 보강 질문 | 의도 |
|---|---|
| "회귀 종류를 구분하는가?" (성능 회귀 / 시각 회귀 / 논리 회귀 / 상태 일관성) | bench 게이트가 시각 회귀 감지 못하는 함정 방어 |
| "인접 테스트가 **같은 호출부** 를 덮는가?" | 간접 보증 범위의 착각 방어 |
| "현 구조에 묶인 판정인가, 리팩터 후 판정인가?" | "Scene 클로저" 같은 구조 의존 비용 평가가 과대계상되지 않도록 |

##### sub-agent 지침 후보

- ROI 5문 체크로 **테스트 생략 판정 시** 추가 검증 루틴 삽입: "테스트 대상 로직이 순수 함수로 추출 가능한 형태인가?" — yes 면 체크 통과와 무관하게 추출 + 테스트 우선

##### CLAUDE.md 실전 교훈 후보 추가 (volt #49 보완)

기존: "주석 계약 vs 구현 drift — 버그 생성원"

추가 명제: **"테스트 생략 판정의 drift — 회귀 생성원"** — 주석 계약이 구현과 drift 하는 문제만이 아니라, **ROI 판정이 실제 회귀 특성과 drift** 하는 문제가 동등하게 위험. 박제 위치 후보: CLAUDE.md 스프린트 계약 6항 직후 "6-a 순수 함수 추출 우선" 하위 항.

## 관련 노트/링크

- astro-simulator ADR: `docs/decisions/20260423-display-relative-scale-unification.md` §Amendments "2026-04-23 — QA 회귀 수정: setTier 대칭 처리 (#313 M2 QA)"
- astro-simulator PR #315 커밋 608e9ab (수정)
- astro-simulator PR #315 QA 코멘트 (회귀 발견): https://github.com/coseo12/astro-simulator/pull/315#issuecomment-4305254780
- astro-simulator QA 재검증 코멘트 (해소 실증): https://github.com/coseo12/astro-simulator/pull/315#issuecomment-4305500528
- astro-simulator #313 완결: https://github.com/coseo12/astro-simulator/issues/313
- volt #49 (주석 계약 vs 구현 drift): https://github.com/coseo12/volt/issues/49
- CLAUDE.md 스프린트 계약 6항 (ROI 5문 체크 원본)
