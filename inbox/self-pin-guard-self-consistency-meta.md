---
title: 자기 박제 가드 self-consistency 메타 검증 — 가드가 자기 자신 검증으로 9~11중 결정적 일치
type: knowledge
source_repo: coseo12/astro-simulator
source_issue: 100
captured_at: 2026-05-17
status: inbox
tags: [self-consistency, meta-verification, defense-in-depth, dogfooding, multi-stage-confirmation]
related:
  - ../notes/cross-validate-plan-mode-bypass-guard.md
  - ../meta/patterns/closing-issues-references-empirical-verify.md
  - ../notes/sub-agent-runtime-ssot-variance.md
---

## 요약

가드 (verify-*.sh / outcome 스키마 / SSoT 박제) 도입 PR 의 self-consistency 입증은 **가드가 자기 자신을 검증**하는 메타 구조로 입증 가능. 5 페르소나 sub-agent (architect → dev → reviewer → qa → 메인) 가 각자 독립으로 동일 outcome 을 산출 → 9~11중 결정적 일치 입증.

## 본문

**패턴 (volt #479 가드 self-consistency 입증 사례)**:

```
PM → architect (Gemini cross-validate 적용 의무) → developer (D4 회귀 가드)
                                                  ↓
                                        reviewer (정적 검증 2,3) → qa (동적 매트릭스 4)
                                                                  ↓
                                                          메인 (outcome.json parse 5)
```

각 단계의 검증 결과:
| 단계 | plan_bypass | bypass_files | rollback_failed |
|------|-------------|--------------|-----------------|
| 1. architect (Gemini 자체 결과) | false | [] | false |
| 2. dev D4 자기 검증 | false | [] | false |
| 3. reviewer 정적 분석 | false | [] | false |
| 4. qa 동적 매트릭스 (4 케이스) | false | [] | false |
| 5. 메인 outcome.json parse | false | [] | false |

**결정적 일치** = 5 단계 × 3 필드 = **15 셀 모두 동일** → 가드 작동 + self-consistency 입증.

**9~11중 메타 변형**:
- 본 PR 검증: 5 단계 × 3 필드 = 15 셀
- 직전 PR (#475, 7 cells), #478 (3 cells), #481 (2 cells), #482 (1 cell) 의 누적 메타 = 28+ 셀

**자기 박제 가드 자기 검증의 의의**:
- "가드가 잡아야 할 상황이 실제로 일어나지 않았는가" 의 강한 증거 (5 독립 검증)
- 단일 검증보다 9~11중 결정적 일치가 self-attestation 함정 차단

**적용 조건**:
- 가드 도입 PR (verify-*.sh / outcome 스키마 / SSoT 박제 등)
- DoD 에 "회귀 가드 시뮬레이션" 명시
- 5 페르소나 sub-agent 워크플로 운영 시

**한계**:
- 모든 5 페르소나가 같은 LLM (Claude) 컨텍스트 공유 → "독립 검증" 의 실질 수준은 모델 단일성에 제약
- Gemini cross-validate 호출이 보강 (다른 모델 시각) 하나 의무 아님
- 메타 측정 도구 (verify-*.sh) 자체가 일관성 부재 시 5 단계 결과가 동일하지 않음 → 메타 측정 도구 자기 적용 안정성 (volt 캡처 별도 후보) 우선 검증

## 관련 노트/링크

- 사례: PR #482 (#479 cross-validate plan-mode 우회 가드, 2026-05-15)
- 누적 self-consistency: PR #472 (4/7) / PR #475 (2/6) / PR #478 (2/3) / PR #481 (2/2) / PR #482 (9중)
- 관련 volt: #99 (cross-validate plan-mode 우회), #93 (auto-close), #58 (sub-agent SSoT variance)
