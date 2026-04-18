---
title: dual PR 변형의 drift 타임라인 — gitflow 선언/현실 불일치 6일간 방치 사례
type: pattern
source_repo: coseo12/harness-setting (v2.12.0 이전 → v2.13.0 복원)
source_issue: 36
captured_at: 2026-04-19
status: refined
tags: [gitflow, dual-pr, drift, branch-strategy, retrospective, single-developer, ai-pair]
related:
  - ../../notes/no-op-adr-handoff-verification.md
  - ../../notes/release-pr-merge-strategy.md
  - ../../notes/gitflow-drift-4tier-classifier.md
  - ../../notes/develop-dual-role-integration-staging.md
---

## 배경/상황

harness-setting (Claude Code AI 페어 프로그래밍 템플릿) 의 브랜치 정리 중 `develop` 이 `main` 보다 **56 커밋 뒤처진 drift** 발견 (2026-04-18, v2.12.0 릴리스 직후). 마지막 develop 업데이트는 2026-04-13 (#57). 이후 6일간 선언 (CLAUDE.md "develop: 통합 브랜치") 과 실제 워크플로 (feature → main 직접 PR) 불일치.

## 내용

#### 1단계 (PR #37~#58, 2026-03-28 ~ 04-12): dual PR 변형

한 작업을 `base=develop` + `base=main` **두 PR 로 각각 머지** 하는 고비용 변형:

```
PR #37 develop → PR #38 main   (같은 내용 2번 머지)
PR #39 develop → PR #40 main
... (#57/#58 까지 11쌍 연속)
```

정석 gitflow (feature → develop → release → main) 가 아닌 변형. release 브랜치 경유 머지 개념 부재.

#### 전환점 (PR #59, 2026-04-15)

harness 확장 작업 (record-adr / update Phase A~C / agents Phase 1~3) 이 4일간 42 PR 로 쏟아지면서 dual PR 유지 비용 포기.

#### 2단계 (PR #59~#99, 4일): main-only (develop 방치)

42 PR 연속으로 `base=main`. develop 업데이트 완전 중단. CLAUDE.md 선언은 그대로.

## 교훈 / 다음에 적용할 점

1. **변형된 gitflow 구현의 구조적 취약성** — dual PR 이라는 고비용 변형은 고빈도 작업 압박 하에서 첫 번째로 희생됨. 정석 gitflow 는 비용 분산 구조
2. **선언/현실 불일치 자동 감지 부재** — 워크플로 변경 시 문서(CLAUDE.md) 갱신 가드가 없어 6일간 drift 발견 못함. `harness doctor` 의 "gitflow 브랜치 정합성" 체크가 이 신호 감지 역할
3. **복원 비용은 drift 기간에 비례** — 56 커밋 fast-forward 로 복구는 가능했으나 이력 파악과 정책 재박제 비용이 큼
4. **1인 + AI 환경 특유 패턴** — 팀 환경이면 동료 리뷰에서 drift 가 조기 발견되지만 AI 는 오히려 drift 를 따라가는 편향. AI 에이전트 세션 시작 시 브랜치 상태 점검 훅이 필수

## 관련 노트/링크

- harness-setting v2.13.0 (ADR: docs/decisions/20260419-gitflow-main-develop.md)
- harness 릴리스 [v2.13.0](https://github.com/coseo12/harness-setting/releases/tag/v2.13.0)
- volt #14 (NO-OP ADR) 와 결은 다르나 "선언 재검증" 공통 원리
