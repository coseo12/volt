---
title: Phase 분리 전략 — 이슈 1개를 3릴리스로 쪼개는 리듬 실측
type: pattern
source_repo: coseo12/harness-setting
source_issue: 30
captured_at: 2026-04-18
status: inbox
tags: [phased-release, sprint-sizing, incremental-delivery, feedback-loop, backward-compatible, minor-release]
related:
  - ../notes/cross-validate-accept-vs-defer.md
  - ../meta/patterns/test-roi-rescope-pattern.md
---

## 요약

완료 기준 5개짜리 이슈를 한 스프린트에 몰아 처리하지 않고 **Phase 분할 점진 릴리스** 로 진행한 실측. Phase 1 (스키마 + 핵심 로직, v2.9.0) / Phase 2 (UX 안내 + 회귀 가드, v2.10.0) 로 쪼개어 각 Phase 가 **독립 릴리스 가능한 관찰 단위** 가 되도록 분리. 결과: 리뷰 부담 분산 + 사용자 관찰 기회 + 설계 재조정 여지.

## 본문

#### 상황

harness [#92](https://github.com/coseo12/harness-setting/issues/92) 는 Gemini 교차검증에서 도출된 후속 이슈로 완료 기준 5개:
1. `.harness/manifest.json` 에 `previousSha256` optional 필드 추가
2. `harness update` 후 매니페스트에 이전 sha256 자동 기록
3. `harness update --check` 가 previousSha256 매치 시 `modified-pristine` 재분류
4. `harness doctor` 해시 정합성 리포트가 previousSha256 매치를 "외부 롤백 의심" 별도 분류
5. merge type 스킵 / managed-block 오탐 방지 테스트 보강

#### 분할 기준

**한 Phase 가 단독으로 관찰 가능한 행동 변화를 만드는가?** 로 판단.

**Phase 1 (1~3번, v2.9.0)** — 매니페스트에 previousSha256 이 기록되고 자가 복구 로직이 동작. 사용자가 외부 롤백 상황에서 `--apply-all-safe` 로 복구되는 것을 **직접 관찰 가능**. 독립 Phase 성립.

**Phase 2 (4~5번, v2.10.0)** — doctor 가 복구 가능 신호를 분리 노출 (UX 개선) + 회귀 가드 테스트. Phase 1 의 로직 위에 얹힌 **가시성 계층**. 독립 Phase 성립.

#### 분할 vs 통합 비교 (실측)

| 지표 | 분할 (실측) | 통합 (가정) |
|------|------------|------------|
| PR 크기 | 약 270줄 + 230줄 | 약 500줄 단일 |
| 리뷰 시간 | 각각 분산 | 한 번에 집중 |
| 릴리스 간 관찰 기회 | 있음 (v2.9.0 후 2시간) | 없음 |
| 설계 재조정 여지 | Phase 2 에서 Phase 1 결과 기반으로 조정 가능 | 일괄 설계로 고정 |
| 롤백 비용 | Phase 단위 독립 | 전체 롤백 |
| backward-compat 검증 | Phase 1 후 독립 검증 | 릴리스 후 일괄 검증 |

#### 적용 가능 조건

- **backward-compatible** 한 단계적 변경일 때 (Phase 1 만 배포되어도 시스템이 정상 동작)
- 각 Phase 가 **완결된 Behavior Change** 집합을 만듦 (중간 Phase 가 부분 구현 상태가 아님)
- 사용자가 **점진 릴리스** 리듬에 동의 (의존성 있는 프로젝트에 주간 단위로 여러 릴리스 허용)

#### 적용 불가 조건

- Phase 간 **필수 의존**: Phase 1 단독 배포 시 시스템이 망가지면 분리 금지
- **파이프라인 변경** 이 모든 Phase 를 통째로 요구 (부분 배포 시 중간 상태가 불안정)
- 사용자가 **단일 메이저 릴리스** 리듬을 선호

#### 이번 케이스의 관찰

- Phase 1 (v2.9.0) 머지 직후 Phase 2 착수까지 시간적 여유는 매우 짧았음 (동일 세션 내). 실무에선 최소 1일 이상 관찰 기간을 두는 것이 권장
- 단일 이슈(#92) 를 Phase 분할로 다루면서도 이슈 자체는 Phase 2 완료 시 한 번에 close (각 Phase 별 sub-이슈 생성은 과설계로 판단)
- CHANGELOG 는 Phase 별 별도 entry (v2.9.0 / v2.10.0) — 독자에게 "왜 두 릴리스로 쪼개졌는지" 가 drift 되지 않도록 각 entry 의 Notes 에 상호 링크 박제

## 관련 노트/링크

- harness [#92](https://github.com/coseo12/harness-setting/issues/92) — 원 이슈 (Phase 1/2 스프린트 계약 댓글 박제)
- harness [PR #94](https://github.com/coseo12/harness-setting/pull/94) — Phase 1 (previousSha256 로직)
- harness [PR #95](https://github.com/coseo12/harness-setting/pull/95) — Phase 2 (doctor 분류 + 회귀 가드)
- harness [v2.9.0 릴리스](https://github.com/coseo12/harness-setting/releases/tag/v2.9.0), [v2.10.0 릴리스](https://github.com/coseo12/harness-setting/releases/tag/v2.10.0)
- volt knowledge 후보: "작은 MINOR 연속 릴리스 리듬" vs "대형 MAJOR 일괄 릴리스" 비교 (추가 관찰 축적 후 knowledge 로 승격 가능)
