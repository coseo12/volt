---
title: reviewer → cross-validate → 반영 → 분리 → 머지 5단계 루프의 세션 효율 패턴 — 9 릴리스 연속 처리 실측
type: report
source_repo: coseo12/harness-setting (2026-04-19 단일 세션, v2.15.1 → v2.19.0 9 릴리스)
source_issue: 44
captured_at: 2026-04-19
status: inbox
tags: [reviewer, cross-validate, pr-loop, release-cadence, follow-up-separation, phase-separation, session-density]
related:
  - ../meta/patterns/cross-validate-after-policy-freeze.md
  - ../notes/cross-validate-accept-vs-defer.md
  - ../meta/patterns/phased-release-split-pattern.md
  - ../meta/patterns/sub-agent-finalization-miss-pattern.md
---

## 배경/상황

2026-04-19 단일 세션에서 harness-setting 에 **9개 릴리스 연속 처리** (v2.15.1 PATCH → v2.16.0 MINOR → v2.16.1 PATCH → v2.17.0 MINOR → v2.18.0 MINOR → v2.19.0 MINOR + 중간 chore 릴리스들). 각 MINOR 마다 동일한 **5단계 루프** 가 반복됐고, 이 패턴이 세션 밀도에도 깨지지 않고 안정적으로 동작.

## 내용

#### 5단계 루프

```
1. reviewer sub-agent 호출 → 정적 리뷰
2. Gemini cross-validate 호출 → 독립 시각
3. 고유 발견/권고 반영 (범위 내)
4. 범위 밖 권고 → 후속 이슈 분리
5. 머지 → chore release prep → main merge commit → tag → release
```

#### 각 단계의 실측 효율

| 단계 | 평균 토큰 | 가치 | 실패 모드 |
|---|---|---|---|
| 1. reviewer | 50-80k | 정적 버그 + 일관성 검출 | approve 남발 시 가치 낮음 |
| 2. cross-validate | 20-40k | 편향 발견 / 고유 관점 | 429 시 폴백 프로토콜 발동 (관찰 0회) |
| 3. 범위 내 반영 | 10-20k | 품질 향상 직접 반영 | 스프린트 비목표 침범 위험 |
| 4. 후속 분리 | 5-10k | 맥락 유실 방지, 미래 자산 | 즉시 생성 안 하면 잊힘 |
| 5. 머지 + 릴리스 | 15-25k | 다운스트림 노출 | PR 체인 꼬임 (develop vs main) |

5단계 루프 당 평균 100-150k 토큰. MINOR 한 개당. 9 릴리스 연속 처리는 토큰 밀도 높음에도 안정적.

#### 반복 패턴의 안정성 요인

1. **5단계 자체가 고정 순서** — 다음 단계가 명확해 결정 피로 최소화
2. **각 단계가 독립 컨텍스트** — reviewer / cross-validate 가 sub-agent 격리로 메인 컨텍스트를 오염시키지 않음
3. **후속 분리 규약 박제** (CLAUDE.md "3단 프로토콜") — 권고 분류 기준이 매번 동일 → 판정 피로 없음
4. **Phase 분리 가능** — 큰 범위는 Phase 1/2/3 으로 자연 분할, 각 Phase 가 5단계 루프 1회
5. **reviewer / Gemini 양쪽이 approve 가 아닌 request_changes / 차단을 꺼리지 않음** — 가짜 approve 보다 차단이 세션 전체 품질 향상

#### 한 세션에서 9 릴리스가 가능했던 추가 요인

- **릴리스 판정 기준이 명확** (CLAUDE.md `### 릴리스` SemVer 규약) → 매 PR 마다 분류 논쟁 없음
- **가이드된 merge 전략** (ADR 20260419-release-merge-strategy) → release PR merge commit + fast-forward develop, 실수 없음
- **auto-close 검증 루틴** (v2.16.0 박제) → release PR 머지 시 이슈 close 자동 + 검증
- **PATCH / MINOR 묶음 릴리스** — follow-up 이슈들 정리할 때 하나씩이 아닌 2~3개 묶어 PATCH 로 (v2.16.1 에서 #118 #114 묶음)

#### 실패 모드 (관찰된 마찰)

- **사용자 확인 과다 요청** (PR #117 Gemini 결과 반영 전 재확인) → 사용자가 "진행 안되는 이유는?" 피드백으로 지적. 이후 과도한 확인 자제
- **reviewer 차단 재발** (PR #134 에서 Phase 2 차단 해소 후 Phase 3 에서 동일 패턴 "wishful documentation" 차단 재현) → 차단 근거가 같은 범주이면 다음 유사 PR 에서 사전 대응 필요
- **chore release PR 에서 CHANGELOG Unreleased → 버전 헤더 누락** → 커밋 메시지 자동화 스크립트 후보

## 교훈 / 다음에 적용할 점

1. **5단계 루프를 PR 템플릿 체크박스로 박제** — 릴리스 체크리스트에 "reviewer ✓ / cross-validate ✓ / 반영 ✓ / 분리 ✓ / 머지" 가시화
2. **차단 근거 분류 사전 대응** — "wishful documentation" 차단이 2회 발생. 다음 유사 PR 은 스프린트 계약에 "프롬프트 레벨 강제 단계 포함" 기준 선명시
3. **사용자 확인 요청 최소화 규약** — 스프린트 계약 승인 받은 후에는 reviewer 권고 반영 / Gemini 고유 발견 반영까지 사용자 재확인 없이 진행. 특별한 설계 결정만 확인
4. **Phase 분리 + MINOR 전용 루프 / PATCH 묶음 루프 차별화** — MINOR 는 5단계 전부, PATCH 묶음은 reviewer + cross-validate 생략 가능 (단, 릴리스 직전 릴리스 PR 에서 1회 일괄 검증)
5. **세션 밀도 상한 관찰** — 한 세션 9 릴리스는 가능했지만 피로 누적. 다음부터는 5~6 릴리스에서 끊고 회고 삽입

## 관련 노트/링크

- volt [#23](https://github.com/coseo12/volt/issues/23) — 박제 직후 cross-validate 루틴
- volt [#29](https://github.com/coseo12/volt/issues/29) — 고유 발견 수용/분리 3단 프로토콜
- volt [#30](https://github.com/coseo12/volt/issues/30) — Phase 분리 전략
- volt [#24](https://github.com/coseo12/volt/issues/24) — sub-agent 마무리 체크리스트
- harness releases [v2.15.1](https://github.com/coseo12/harness-setting/releases/tag/v2.15.1) ~ [v2.19.0](https://github.com/coseo12/harness-setting/releases/tag/v2.19.0) (9 릴리스)
- CLAUDE.md `### 릴리스` / `### 교차검증` / `### sub-agent 검증 완료 ≠ GitHub 박제 완료`
