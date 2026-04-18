---
title: PM sub-agent multi-turn 라운드 이탈 — 메인이 라운드 간 매트릭스 일관성 검증 필수
type: pattern
source_repo: coseo12/astro-simulator
source_issue: 34
captured_at: 2026-04-18
status: inbox
tags: [pm-agent, multi-turn, sub-agent, context-loss, sendmessage, orchestration, harness]
related:
  - ../meta/patterns/sub-agent-finalization-miss-pattern.md
---

## 배경/상황

태양계 정밀화 로드맵 설계를 PM sub-agent에 위임하여 3라운드 적응적 질답 진행. 라운드 2에서 사용자가 **"권고안 A (9개 Phase, 위성·고리·궤도정밀·배포·소행성·기술부채)"**를 선택. 이어 SendMessage로 라운드 3 호출 시 "권고 A 확정"을 지시했으나, PM sub-agent가 **라운드 2 합의 내용을 이탈**하여 전혀 다른 Phase 매트릭스(J2/Yarkovsky/중력파/교육 등)를 제시.

## 내용

**관찰된 이탈 패턴**:

| 라운드 2 권고 A | 라운드 3 sub-agent 제시 |
|---|---|
| P8 내행성계 위성 (화성 2체+달) | P8 태양계 완전성 (왜행성+주요 위성) |
| P9 목성계 (갈릴레이 4+희미한 고리) | P9 궤도 섭동 고도화 (J2/J4) |
| P10 토성계 (10+위성+A/B/C 링) | P10 비중력 가속도 (Yarkovsky) |
| P11 천왕성계 (5위성+11링) | P11 항성 배경·성도 |
| P12 해왕성계 | P12 외계 행성계 |

**원인 추정**:
- SendMessage agentId 지시했으나 실제 반환 agentId가 다름 — 컨텍스트 이어받기 실패
- sub-agent가 "P8~P16 로드맵 설계" 전체 목적만 유지하고 라운드 2 매트릭스 세부를 잃음
- 사용자 답변 (Q2=(c) 30+ 위성 / Q3=전부 4행성 고리)과의 연결이 끊김

**감지 방법**:
- 메인 오케스트레이터가 라운드 간 출력을 직접 대조 — 자동 diff 없음
- Phase 매트릭스의 주요 키워드(위성명/고리명) 일치 여부 수동 확인
- 이번 케이스: "사용자 Q2/Q3 답변과 라운드 3 매트릭스가 태주제 다름" 검출

**해결**:
- 라운드 3 결과 폐기, 라운드 2 권고 A를 메인이 직접 메모리에 박제 (`project_p8_p16_roadmap.md`)
- 사용자에게 불일치 보고 + 선택 재확인 → A/B/C 선택지 제시
- 사용자 "A 진행" 승인으로 라운드 2 매트릭스 확정

## 교훈 / 다음에 적용할 점

1. **PM sub-agent multi-turn 세션은 라운드마다 컨텍스트 격리 발생 가능** — SendMessage `to:` 지시만으로 전 라운드 세부 내용이 복원되지 않는 경우가 있음. 특히 복잡한 매트릭스/테이블 박제 시 주의.

2. **메인 오케스트레이터의 라운드 간 검증 책임**:
   - 각 라운드 출력에서 **핵심 keyword 목록** 추출 (Phase 제목 / DoD 수치 / 의존 관계 등)
   - 다음 라운드 출력과 대조하여 이탈 감지
   - 이탈 시 즉시 사용자에게 보고 + 이전 라운드 재확인

3. **라운드 이탈 방지 프롬프트 패턴**:
   - SendMessage 호출 시 **"이전 라운드 매트릭스를 그대로 유지하고 다음 단계만 추가"** 를 명시
   - 이탈 위험 내용은 **본문에 직접 인라인** (요약 2~3줄로라도)
   - "권고안 A" 같은 참조 레이블 대신 원문 매트릭스 전체 재첨부

4. **라운드 3 결과가 라운드 2 합의 이탈해도 가치 자산으로 보존**:
   - 이번 이탈 결과의 "J2/Yarkovsky/중력파/교육" 주제는 실제로 P17+ 확장 아이디어로 유효
   - 메모리 `project_p8_p16_roadmap.md` §참고에 "라운드 3 이탈 내용 = P17+ 후보" 로 박제
   - sub-agent 이탈을 손실이 아닌 보너스 자산화

5. **harness 개선 후보**:
   - PM 스킬 문서에 "multi-turn 라운드 이탈 검증" 체크리스트 추가
   - Agent SendMessage 결과를 메인 컨텍스트에 메타(agentId 일치 확인) 자동 표시
   - 라운드 N의 출력을 라운드 N+1 프롬프트에 자동 snapshot 첨부하는 휴리스틱 탐색

## 관련 노트/링크

- capture-merge 후속 이슈 (예정): astro-simulator repo memory `project_p8_p16_roadmap.md`
- volt #24 (선행 패턴): sub-agent 검증 완료 ≠ GitHub 박제 완료 — 유사 "sub-agent 출력 신뢰 한계" 계열
- CLAUDE.md CRITICAL #2 — 모호한 지시 사전 범위 확인
