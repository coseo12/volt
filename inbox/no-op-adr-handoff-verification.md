---
title: 회고 인계 항목은 착수 시점에 실측 재검증 — NO-OP ADR 패턴
type: knowledge
source_repo: coseo12/astro-simulator
source_issue: 14
captured_at: 2026-04-16
status: inbox
tags: [pattern, adr, retrospective, verification]
related:
  - ../meta/patterns/sprint-contract-retrospective.md
---

## 요약

이전 마일스톤 회고가 인계한 "수정 필요 항목"이 다음 마일스톤 착수 시점에서는 이미 해결되어 있는 경우가 있다. 구현에 바로 들어가기 전에 실측으로 전제를 재검증하고, 해소됐다면 구현 대신 **NO-OP ADR**로 종결한 뒤 회귀 가드만 박제한다. 이 패턴은 표면적 변경과 표면적 테스트 증가를 방지한다.

## 본문

**트리거 조건**

- 이전 회고의 "다음 인계" 항목을 작업 범위로 받음
- 회고 작성 시점 이후 환경/코드가 바뀌었을 가능성이 있음 (Chrome 버전 업, 어댑터 변경, 의존 라이브러리 업데이트 등)

**수행 절차**

1. 작업 착수 전 현재 코드가 이미 목표 동작을 하는지 실측 (브라우저/bench/테스트)
2. 이미 만족하면 구현 대신 `docs/decisions/YYYYMMDD-<topic>-no-op.md` ADR 작성
3. ADR 내용: 후보 비교표 / 실측 결과 / "왜 변경 NO-OP인가" 이유 / 재검토 조건
4. **대신 회귀 가드 박제** — 실측으로 확인된 동작이 퇴행하지 않도록 회귀 검증 스크립트 추가

**구체 사례**

- P3 회고 인계: "Babylon useWebGPU 명시 필요"
- P4 착수 실측: 현재 engine-factory가 이미 `new WebGPUEngine()` 직접 생성 + 사전 adapter 판별로 해결된 상태
- 결정: EngineFactory 전환 NO-OP. 대신 `scripts/browser-verify-webgpu.mjs` 신규 회귀 가드 추가 (5/5 통과)
- 문서: `docs/decisions/20260416-engine-factory-no-op.md`

**왜 이 패턴이 중요한가**

- AI는 인계받은 항목을 "해야 할 일"로 과신하는 경향 → 불필요한 리팩토링 발생
- 실측 없는 전제 재사용은 표면적 변경을 늘리고 회귀 리스크만 키움
- NO-OP 결정도 근거(ADR)가 남으면 다음 마일스톤에서 재발굴 시 빠르게 기각 가능

## 참고

- Sprint Contract 루틴 (CLAUDE.md)
- Claude Code CRITICAL #2 "모호한 지시 사전 확인"과 상호보완 — 이 경우는 "명확한 지시를 받았지만 실측으로 범위 축소" 사례
