---
title: 회귀 가드 시뮬레이션 3중 검증 패턴 — positive→negative→recovery 본 PR 내 박제
type: pattern
source_repo: coseo12/astro-simulator
source_issue: 96
captured_at: 2026-05-17
status: refined
tags: [ci-guard, regression-test, drift-detection, dod-verification, in-pr-simulation]
related:
  - ../../notes/downstream-measurement-final-guard.md
  - ../../notes/debug-script-over-static-analysis.md
  - ../../notes/reviewer-qa-gate-skip-headless-self-pass.md
  - ../../notes/bash-script-isolation-dynamic-test.md
  - ../../notes/work-order-grep-backtick-fidelity.md
  - ../../notes/scattered-constant-data-meta-ssot.md
---

## 배경/상황

SSoT drift 자동 가드 (verify-*.sh + CI step) 같은 negative-test 성격의 가드는 **positive PASS 만으로는 가드 작동을 보장하지 못한다**. 격리된 단위 테스트로 가드 로직 자체는 검증되더라도, "실제 CI 파이프라인이 drift 를 차단하는가" 는 **별개 검증 단위**. PR DoD 에 "회귀 가드 시뮬레이션" 명시하는 사례가 많지만 (이슈 본문에 명시), 실측 패턴이 정형화돼 있지 않아 매번 다른 방법으로 검증됨.

## 내용

**3중 검증 패턴** — 한 PR 내에서 가드 작동을 3 상태로 입증:

| 단계 | 행위 | 기대 CI 결과 | 입증 |
|------|------|--------------|------|
| 1. **positive** | 정상 코드 push | gate step **SUCCESS** | 가드가 정상을 정상으로 분류 |
| 2. **negative** | 의도적 drift commit push (예: SSoT 박제 1 글자 변경) | gate step **FAILURE** | 가드가 drift 를 실제 차단 |
| 3. **recovery** | drift commit revert push | gate step **SUCCESS** 회복 | 가드가 false-positive 가 아니며 정상 회복 가능 |

**astro-simulator PR #496 실측 (2026-05-17)**:

- positive: 초기 commit `74ff457` (5 페르소나 SSoT 정합) → harness-guards SUCCESS ✓
- negative: drift commit `cc0405c` (reviewer.md 의 `#471` → `#999`, SSoT 키 외 부수 텍스트만 변경 — 가드의 sort -u 동일성 검증 발화) → harness-guards **FAILURE** ✓
- recovery: revert commit `24188bf` (`git revert HEAD`) → harness-guards SUCCESS ✓

**격리 동적 테스트와의 직교성**:
- 격리 동적 테스트 (mktemp + AGENT_DIR override) 는 **스크립트 로직** 검증
- 3중 시뮬레이션은 **CI 통합 + hashFiles 조건 + 실제 차단** 검증
- 둘 다 통과해야 가드 작동이 완전 입증됨 (둘 중 하나만 통과 시 어딘가에 함정 잔존)

**비용**:
- CI run 2회 추가 (drift push + revert push)
- PR 머지 전 임시 commit/revert 노이즈 → squash 머지로 자연 흡수
- 약 5~15분 추가 (CI 큐 시간 의존)

**적용 조건**:
- negative-test 성격의 신규 가드 (verify-*.sh + CI step)
- DoD 에 "회귀 가드 시뮬레이션" / "drift 차단 실측" 명시된 PR
- 도메인 위험 큰 가드 (SSoT drift, 보안, ADR 호환성 등)

## 교훈 / 다음에 적용할 점

- **신규 verify-*.sh + CI step PR DoD 에 본 패턴 명시 의무** — "positive PASS 만으로 충분하다" 는 self-attestation 함정 차단
- **PR 커밋 컨벤션**:
  - drift commit: `test(infra): [#N] 의도적 drift commit — CI 차단 발화 실측용 (revert 예정)`
  - revert commit: `git revert --no-edit` (자동 메시지)
- **타이밍**: positive CI PASS 확인 후 drift push (negative 만 단독 push 면 PR 머지 차단 상태로 오인 위험)
- **격리 동적 테스트와 직교 박제** — 둘 다 DoD 에 명시. 한쪽만 있으면 다른 쪽 함정 미감지
- harness real-lessons 박제 후보 — "빌드 성공 ≠ 동작" 의 negative-test 확장 ("guard PASS ≠ guard 작동")

## 관련 노트/링크

- 사례: PR #496 (#480 5 페르소나 SSoT drift 자동 가드, 2026-05-17)
- 직교 관련: harness `verify-agent-ssot.sh` (#145) — 동일 패턴으로 도입 시 본 3중 시뮬레이션 누락
- 관련 volt: #60 (다운스트림 실측), #67 (debug 스크립트 실측), #77 (메인 단계 게이트)
