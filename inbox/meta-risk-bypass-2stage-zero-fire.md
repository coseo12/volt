---
title: 메타 위험 우회 패턴 — 가드 도입 PR 에서 architect + dev D4 2회 연속 위험 발화 0
type: knowledge
source_repo: coseo12/astro-simulator
source_issue: 112
captured_at: 2026-05-17
status: inbox
tags: [meta-risk, guard-self-test, architect-cross-validate, dev-d4, dogfooding-success]
related:
  - ../notes/cross-validate-plan-mode-bypass-guard.md
  - ../notes/self-pin-guard-self-consistency-meta.md
  - ../notes/meta-measurement-tool-self-application.md
---

## 요약

가드를 도입하는 PR 의 architect 단계 cross-validate 호출 + dev D4 회귀 가드 자기 검증 — **2 단계 모두 같은 가드가 자기 자신을 검증**. 이 메타 위험 (Gemini plan-mode 우회 사고 재발) 이 2회 연속 발화 0 → 가드 작동 + 메타 안정성 입증 패턴.

## 본문

**사례 (PR #482 #479 cross-validate plan-mode 우회 가드, 2026-05-15)**:

PR #479 는 cross-validate plan-mode 우회 사고 자체를 가드 박제로 차단하는 PR. 가드 = `scripts/cross_validate.sh` (snapshot+hash 비교 + 자동 롤백 + outcome 스키마 확장).

**메타 위험 시퀀스**:
1. architect 단계: Gemini cross-validate 호출 (정상 워크플로)
   - **새 가드가 architect 의 cross-validate 호출 자체를 감시**
   - 만약 Gemini 가 plan-mode 우회 사고 (예상 외 파일 수정) 일으키면 가드가 발화 (`plan_bypass: true`)
   - 발화 시 자동 롤백 → outcome.json 에 박제 → 메인 오케스트레이터가 검출 + 사용자 보고
2. dev D4 단계: 새 가드의 회귀 가드 자기 검증
   - **dev 가 자기 자신을 cross-validate 호출**로 검증 (`plan_bypass=false` 기대)
   - 만약 신규 가드 코드가 plan-mode 우회 일으키면 가드가 자기 자신 감시

**2회 연속 발화 0 결과**:
- architect 단계 outcome: `plan_bypass=false, bypass_files=[], rollback_failed=false`
- dev D4 단계 outcome: `plan_bypass=false, bypass_files=[], rollback_failed=false`
- → **메타 위험이 2회 모두 발화하지 않음** = 가드가 자기 자신을 잘 검증

**의의**:
- 가드 도입 PR 의 self-test 메커니즘 일반화
- "가드가 자기 PR 에 적용될 때 메타 위험이 발화하지 않는가" 의 자기 검증 패턴
- 메타 위험 발화 시 즉시 가드 박제 무산 (자기 자신 검증 fail → 가드 신뢰도 0)

**자기 박제 가드 self-consistency 와의 관계**:
- volt #100 (self-consistency 9~11중 메타) — 5 페르소나 가 동일 outcome 산출 확인
- 본 패턴 — 2 단계 (architect + dev D4) 가 동일 outcome 산출 (`plan_bypass=false`) 확인
- 둘 직교: self-consistency = 가로축 (5 페르소나), 메타 위험 우회 = 세로축 (2 단계 같은 가드 호출)

**일반화 — 가드 도입 PR 의 메타 위험 우회 의무**:
- 가드 도입 PR 의 DoD 에 "architect cross-validate + dev D4 자기 검증 outcome 박제" 의무
- 2 단계 outcome 결정적 일치 ≠ false → 가드 박제 무산 (사고 박제 + 후속 이슈 분리)
- 본 패턴은 메타 위험 (가드가 자기 PR 에 적용되는 시점) 자체를 명시 박제하는 첫 사례

**적용 조건**:
- cross-validate (또는 동종 외부 도구 호출) 가드 도입 PR
- 가드가 자기 자신 검증 메커니즘 가짐 (snapshot + outcome 등)
- 메인 오케스트레이터가 outcome.json parse 의무 가짐

## 관련 노트/링크

- 사례: PR #482 (#479 cross-validate plan-mode 우회 가드, 2026-05-15)
- 관련 volt: #99 (cross-validate plan-mode 우회 가드), #100 (self-consistency 9~11중 메타), #109 (메타 측정 도구 자기 적용 안정성)
