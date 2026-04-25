---
title: PM multi-turn DoD 구조 drift 재현 — volt #34 반복 패턴 확증 (교훈 3)
type: knowledge
source_repo: coseo12/astro-simulator
source_issue: 76
captured_at: 2026-04-25
status: inbox
tags: [pm-agent, multi-turn, dod-drift, sub-agent-orchestration, matrix-consistency, volt-34-followup]
related:
  - ../meta/patterns/pm-sub-agent-multi-turn-round-drift.md
  - ../meta/patterns/sub-agent-finalization-miss-pattern.md
  - ../notes/ssot-json-field-sign-misread.md
---

## 요약

PM sub-agent multi-turn 라운드 전환 시 원본 DoD 매트릭스가 재해석되어 **특정 DoD 가 다른 DoD 대상으로 재배치되면서 누락** 되는 현상 재현. astro-simulator P11-B.2 PM 재계약에서 원본 D5 (Osculating 관찰 리포트) 가 D3b (screenshot diff) 의 7 moon 대상으로 재배치되어 **원본 D5 사라짐**. volt #34 "sub-agent multi-turn 라운드 이탈" 이 단일 사례 아닌 **반복 패턴** 임을 확증.

## 본문

#### 현상

astro-simulator #289 P11-B.2 PM 재계약 중:

**라운드 1 (PM 초안)** — 원본 DoD 3건:
- D3b: browser-verify-lod 9 조합 screenshot diff < 15%
- D4: bench 5 scenario 회귀율 < 5%
- D5: #247 Osculating 관찰 리포트 (`docs/benchmarks/p11-b-osculating-observation-*.md`)

**사용자 응답**: Q1=A, Q2=A, Q3=A, **Q4=C** (전체 위성 Osculating 관찰), Q5=A

**라운드 2 (PM 최종 계약 초안)** — 재구조화 발생:
- D3b: `state.transitioning` 플래그 도입 (← 완전 교체)
- D4: 9 조합 screenshot diff (7 moon × 9) (← D3b 내용이 D4 로 이동 + Osculating 이 screenshot 대상으로 변형)
- D5: bench 회귀 가드 (← 원래 D4 내용이 D5 로)
- **원본 D5 Osculating 관찰 리포트 사라짐**

메인 오케스트레이터가 volt #34 패턴으로 감지. 원본 DoD 복원 + Q4=C 의 관찰 분량 (7 moon vs 4 moon + smoke 3) 만 사용자 승인 후 재조정.

#### 근본 원인

- Multi-turn 라운드 전환 시 **sub-agent 가 사용자 응답을 받아 매트릭스를 재해석** — 각 DoD 의 파라미터만 바꾸는 것이 아니라 DoD 자체를 재배치
- 원본 DoD ID (D3b/D4/D5) 가 **라운드 2 에서 의미 변경** — ID 는 유지되나 내용 drift
- PM 프롬프트에 "DoD 구조 유지, 파라미터만 변경" 제약 부재

#### 교훈 (volt #34 재현 확증)

- Multi-turn PM/architect/designer 라운드는 **원본 매트릭스 유지 의무**. 사용자 응답은 각 항목의 파라미터 (수치 / 선택지 / 경계) 만 조정해야
- 메인 오케스트레이터는 라운드 N+1 출력을 **라운드 N 의 매트릭스 키워드 (DoD ID / 산출물 경로 / 핵심 수치) 로 대조 검증 의무**
- PM sub-agent 프롬프트에 명시 필수:
  - "원본 DoD 재구조화 금지"
  - "사용자 응답은 각 DoD 의 파라미터만 바꾼다"
  - "DoD ID 는 원본 의미 유지, 내용 치환 금지"
- 라운드 N+1 출력에 원본 DoD 목록 **변경 전/후 diff** 명시 요구 (sub-agent 가 자기 검증 가능하도록)

#### volt #34 와의 연결

- volt #34 "sub-agent multi-turn 라운드 이탈 — 매트릭스 일관성 검증" 에서 제시된 예방 루틴 (키워드 추출 + 라운드 전후 대조) 이 본 사례에서 실제 적용되어 성공
- 본 사례는 volt #34 가 **1회성 교훈이 아닌 반복 발생 패턴** 임을 확증 — 에이전트 프롬프트에 제약 명시 없이는 재발 가능성 높음

## 관련 노트/링크

- volt #34 (sub-agent multi-turn 라운드 이탈 — 매트릭스 일관성 검증)
- volt #24 (sub-agent 검증 완료 ≠ GitHub 박제 완료)
- astro-simulator#289 (P11-B PM 재계약 이슈)
- astro-simulator PR #322 (최종 복원된 DoD 로 머지됨)
