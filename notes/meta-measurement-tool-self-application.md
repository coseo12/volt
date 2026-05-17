---
title: 메타 측정 도구 자기 적용 안정성 — verify-*.sh 가 자기 PR 에 적용될 때 self-consistency 결정적 일치
type: knowledge
source_repo: coseo12/astro-simulator
source_issue: 109
captured_at: 2026-05-17
status: refined
tags: [meta-measurement, self-application, deterministic-consistency, dogfooding, guard-introspection]
related:
  - ./sub-agent-runtime-ssot-variance.md
  - ./self-pin-guard-self-consistency-meta.md
  - ../meta/patterns/closing-issues-references-empirical-verify.md
  - ./meta-risk-bypass-2stage-zero-fire.md
---

## 요약

verify-*.sh 류 가드를 도입하는 PR 자체가 본 가드의 검증 대상에 해당될 때, 가드는 자기 자신을 측정 → 결정적 일치 필요. 측정 일관성 부재 시 PR 검증이 비결정적 → 가드 무력화. **메타 측정 도구의 자기 적용 안정성 = 결정적 일치** 가 첫 번째 SSoT.

## 본문

**문제 (PR #472, #475, #478, #481, #482 누적 메타 28 셀)**:

verify-*.sh 가드는 5 페르소나 sub-agent 의 각 단계에서 동일 결과를 산출해야 함:

| PR | 메타 측정 항목 | 일치 셀 / 총 셀 |
|----|----------------|------------------|
| #472 | DoD #2 measurement | 4/7 |
| #475 | PR template 7 체크박스 | 2/6 |
| #478 | create-pr drift | 2/3 |
| #481 | 5 페르소나 SSoT | 2/2 |
| #482 | #479 cross-validate plan-mode | 1/1 (9중 메타) |

**누적**: 28+ 셀 중 모두 결정적 일치 → 메타 측정 도구 자기 적용 안정성 입증.

**측정 일관성 부재 시 발생**:
- 5 페르소나가 같은 가드를 다르게 해석 (예: 같은 SSoT 키를 architect 가 grep -F, dev 가 awk, reviewer 가 정규식 으로 해석 → 결과 셀 비결정)
- 가드 결과가 비결정적 → CI 가 우연 PASS / FAIL 반복 → 가드 자체 신뢰도 폐기

**해결 (메타 SSoT 의무화)**:
1. 가드 도입 PR 의 architect/dev/reviewer/qa 가 **동일 명령** 호출 의무 (예: `bash scripts/verify-X.sh` 만 사용, 변형 금지)
2. outcome.json 스키마 SSoT 의 5 페르소나 .md 동일성 (volt #58 sub-agent SSoT variance 의 메타 적용)
3. 메타 측정 결과를 PR 본문에 표 형태 박제 → "9중 메타 결정적 일치" 가시화

**자기 박제 가드의 의의**:
- 가드가 자기 자신 검증 → "가드 = SSoT" 의 메타 일관성
- 메타 측정 도구 (verify-*.sh) 가 가드 일관성 측정 → 메타-메타 검증 (PR diff 가 가드 자체에 영향 미치는지 확인)
- 6+ 회차 누적 시 패턴화 — N=3 의 Rule of Three 보다 메타 안정성 의무 5+ 회 누적이 더 강한 증거

**적용 조건**:
- verify-*.sh 도입 PR
- 5 페르소나 sub-agent 워크플로 운영
- 메타 측정 결과 셀 ≥ 2 (단일 셀은 결정적 일치 자명함)

## 관련 노트/링크

- 사례: PR #472, #475, #478, #481, #482 누적 메타 (2026-05-13 ~ 2026-05-15)
- 관련 volt: #58 (sub-agent SSoT variance), #100 (self-consistency 9~11중 메타), #93 (auto-close)
