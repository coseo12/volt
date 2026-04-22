---
title: cross-validate 수치 DoD 교정의 self-contradiction — 실측 sanity check 누락으로 2차 수정
type: report
source_repo: coseo12/astro-simulator
source_issue: 66
captured_at: 2026-04-22
status: inbox
tags: [cross-validate, DoD, self-contradiction, sanity-check, implementation-gap]
related:
  - ../meta/patterns/cross-validate-after-policy-freeze.md
  - ../notes/cross-validate-accept-vs-defer.md
  - ../notes/external-tool-claim-empirical-verify.md
  - ../notes/downstream-measurement-final-guard.md
  - ../notes/debug-script-over-static-analysis.md
---

## 배경/상황

P11-A Floating Origin 스프린트 계약 박제 직후 루틴(volt #23) 으로 Gemini cross-validate 실행. Gemini 가 **§2 "DoD β 기준 교정"** 에서 다음 지시:

> "현재 focus 중인 대상체 + 카메라 좌표 절대값 ≤ 1e5 m"

이를 architect / developer 단계에서 그대로 구현 반영 (PR [#291](https://github.com/coseo12/astro-simulator/pull/291) 초기 커밋). QA 1차는 "조건부 통과" 판정했으나 후속 drift 조사에서 **교정안 자체가 물리적으로 실현 불가능한 self-contradiction** 임이 드러남.

## 내용

**모순의 실체**:

scientific 모드는 P10 Fact-First 원칙에 따라 실제 거리(수억 km)를 렌더. 사용자가 목성을 focus 하면 카메라는 focus body 를 관찰하기 위해 **수 AU 떨어진 위치**에 배치됨 (fact 유지). 이는 물리적 시각화 필수.

Floating Origin 이 focus body 로 origin shift 후:
- focus body local ≈ 0 ✓
- **camera local = camera_world − origin = 수 AU = 수천억 m**
- "카메라 local ≤ 1e5 m (100 km)" 제약은 **달성 불가능**

실 dev 서버에서 매 프레임 `console.error` 쏟아짐:
```
[floating-origin] camera local 좌표 초과 (≥1e5m):
  551609191979.78, 267067450948.42, 35002965518.79
```

**파급**:

1. 설계 단계 (architect): cross-validate 교정안을 근거로 ADR §5 DoD β 를 "focus + 카메라" 병행으로 박제
2. 구현 단계 (developer): ADR 계약대로 assert 구현 (`packages/core/src/scene/solar-system-scene.ts:527-554`)
3. QA 1차: DoD β local=0m 만 확인 (focus body 기준), 카메라 assert 실패는 headless stdout 에서 관찰 못 함
4. drift 조사 중 실 dev 서버 debug 스크립트로 노출 → 2차 수정 (커밋 `aefe5e5`) 로 카메라 조건 제거 + ADR §Amendments 정정

**근본 원인**:

cross-validate 가 "jitter 해소를 위해 작은 local 좌표가 필요하다" 는 일반론에서 "모든 관측 대상(body + 카메라)" 으로 범위를 **무비판 확장**. 실 렌더 파이프라인의 현실 (mesh 만 jitter 소스, 카메라는 projection 에서 별도 처리) 을 고려 못 함. 우리는 이를 "수치 DoD 교정 = 엄격할수록 안전" 이라는 직관으로 수용.

## 교훈 / 다음에 적용할 점

**규약 제안: cross-validate 수용 직전 실측 sanity check**

volt #23 "수용 vs 후속 분리 3단 프로토콜" 을 다음처럼 보강:

0. **(신규) 수용 전 실측 sanity check** — 교차검증이 제안한 **수치 DoD 재정의** 는 1회 실 환경 실행 또는 단위 테스트 snippet 으로 자가모순 확인 선행:
   - "이 조건이 정상 동작에서 만족 가능한가?" 를 현존 시스템에서 직접 확인
   - 실측 없이 DoD 박제 → 구현 착수 → 2차 수정 비용은 초기 30분 실측 대비 훨씬 큼
1. 합의 선별 (기존)
2. 고유 발견의 범위 체크 (기존)
3. 분리 시 박제 규칙 (기존)

**적용 포인트**:

- cross-validate 스킬 (`.claude/skills/cross-validate/SKILL.md`) "결과 분석" 섹션에 단계 0 추가
- architect 에이전트: ADR 작성 시 수치 DoD 인용 전 "실측 sanity check" 완료 여부 체크
- 일반 원칙: **"엄격할수록 안전" 직관은 물리/도메인 제약을 무시할 때 오작동한다** — 모든 수치 제약은 실현 가능성 먼저

**추가 관찰**:

AI (Gemini + Claude 모두) 는 "엄격한 DoD" 에 편향되어 self-contradiction 을 간과하는 경향. 교차검증도 이 편향을 공유하므로 **실측**이 유일한 가드.

**cross-ref**:

- volt #23 (수용 vs 분리 3단 프로토콜)
- volt #51 (cross-validate 오탐 2건 — 외부 툴 실측 필수)
- volt #60 (다운스트림 실측이 최종 가드)
- 원 PR: coseo12/astro-simulator#291 (커밋 `aefe5e5` 2차 수정)
- ADR Amendment: `docs/decisions/20260422-floating-origin.md` §Amendments 2026-04-22
