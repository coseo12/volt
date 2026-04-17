---
title: 신규 함수 작성 전 기존 유사 함수 탐색 누락 — 중복 코드 방지
type: report
report_type: feedback
source_repo: coseo12/astro-simulator
source_issue: 21
captured_at: 2026-04-17
status: refined
tags: [ai-feedback, code-duplication, workflow]
related: []
---

## 배경/상황

P5-A(일반상대론) 작업 중 Kepler 궤도 요소에서 position + velocity를 계산하는 `stateVectorAt()` 함수를 `physics/kepler.ts`에 새로 작성. 테스트(4개)까지 작성 완료 후, 동일 기능의 `orbitalStateAt()` 함수가 `physics/state-vector.ts`에 이미 존재하고 `nbody-engine.ts`에서 사용 중임을 발견.

## 내용

**시도**: `kepler.ts`에 50줄 + 테스트 70줄 작성
**발견**: `state-vector.ts`에 동일 함수 이미 존재 (다른 수학적 접근이지만 결과 동일)
**결과**: 중복 코드 + 테스트 전부 삭제. 기존 `orbitalStateAt()` 재사용.
**소요**: ~15분 낭비 (작성 + 테스트 + 발견 + 삭제)

**근본 원인**: AI가 "필요한 함수가 없다"고 가정하고 바로 작성에 들어감. `grep "velocity\|stateVector"` 같은 탐색을 사전에 하지 않음. `import { orbitalStateAt }` 라인이 같은 패키지의 다른 파일에 있었으나 확인 안 함.

## 교훈 / 다음에 적용할 점

**agent/skill 개선**:

1. **신규 함수 작성 전 의무 탐색** — `Grep`으로 함수명·핵심 키워드 검색. 예: `stateVector`, `velocity.*orbital`, `position.*kepler` 등.
2. **같은 패키지의 index.ts export 확인** — `packages/core/src/physics/index.ts`만 봐도 `state-vector.ts` export가 보였을 것.
3. **"이미 있을 수 있다"는 가설을 먼저** — AI는 "없다"고 가정하고 바로 구현하는 편향이 있음. 특히 이전 마일스톤에서 이미 구현된 유틸리티는 높은 확률로 존재.
4. **developer agent에 사전 탐색 절차 추가** — "신규 헬퍼/유틸리티 작성 전 기존 유사 함수 grep" 규칙을 `.claude/agents/developer.md`에 추가 검토.
