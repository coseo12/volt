---
title: 신규 데이터 ≠ 신규 코드 — ADR 예측 재현으로 추상화 건강성 실증
type: knowledge
source_repo: coseo12/astro-simulator
source_issue: 47
captured_at: 2026-04-19
status: refined
tags: [adr, abstraction, pattern, data-driven, parent-child, json-schema, prediction]
related:
  - ../meta/feedback/new-function-prior-search-miss.md
---

## 요약

ADR 설계 시 "신규 기능 = 데이터 추가만, 코드 라인 추가 0" 을 예측 박제하고 머지 시점에 실측으로 재현 확인하면, 기존 추상화가 올바르게 설계됐는지 구체 증거로 검증된다. volt #21 "신규 함수 ≠ 신규 구현" 의 **데이터 파생 형태**. parentId 체인 / 플러그인 레지스트리 / 라우팅 테이블 같은 계층적 데이터 추상화 시나리오에서 광범위 적용 가능.

## 본문

**사례**: astro-simulator P8 ADR (`docs/decisions/20260419-satellite-orbit-hybrid.md`) 에 다음 예측 박제.

> 달은 이미 `solar-system.json` 에 정의되어 있고 (`parentId: earth`), `updateAtKepler` 경로로 이미 렌더 작동 중. **포보스/데이모스만 JSON 추가하면 TS 렌더 코드 라인 변화 0**.

3-PR 분할 구현 중:
- PR-1 (#248): `solar-system.json` 에 포보스/데이모스 2 엔티티 추가 (`parentId: mars`)
- PR-3 (#252): **sim-canvas 코드 변경 0 줄** — 실측 재현

기존 추상화가 자동 수용한 3 경로:
1. `updateAtKepler(bodyId, t)` — scene graph 재귀 호출
2. `CelestialTree` `childrenOf(parentId)` — 사이드패널 계층 렌더
3. `solar.meshes.get(bodyId)` — focus 카메라 전환

즉 기존 3 계층 모두 `parentId` 를 정확히 **데이터로만 참조**하고 있었고, 화성/지구 하드코딩이 0 건이었음.

**일반화 패턴**:

1. ADR 작성 시 "코드 라인 0" 을 Concrete Prediction 으로 박제 — 추상화 건강성 사전 테스트
2. 실구현 PR 의 CI 에 `git diff --stat <추상화 계층 경로>` 가드 추가 가능 — "계층 수정 없이 기능 추가 가능" 을 자동 검증
3. 예측 실패 (=계층 수정 필요) 시: (a) 추상화가 부족하다는 신호 — 먼저 리팩토링 (b) 또는 예외 케이스 인정 후 ADR Amendment 박제

**volt #21 확장 관계**:
- volt #21: "신규 함수 쓰기 전 이미 있는지 Grep 확인" — 코드 차원 중복 회피
- 본 패턴: "신규 데이터 추가 전 계층이 이미 수용 가능한지 확인" — 데이터 차원 추상화 건강성 검증
- 둘 다 "건강한 추상화는 최소 delta 로 확장 가능" 을 공유

**적용 가능 시나리오**:
- 플러그인/미들웨어 레지스트리 — 신규 핸들러 추가가 core 수정 없이 가능한가?
- 라우팅 테이블 — 신규 라우트 추가가 라우터 코드 수정 없이 가능한가?
- 스키마-주도 UI (form builder, dashboard) — 신규 필드 추가가 렌더 엔진 수정 없이 가능한가?
- 번역 테이블 / 국제화 — 신규 언어 추가가 i18n 라이브러리 수정 없이 가능한가?

## 관련 노트/링크

- ADR 원문: https://github.com/coseo12/astro-simulator/blob/main/docs/decisions/20260419-satellite-orbit-hybrid.md
- PR-3 실측: https://github.com/coseo12/astro-simulator/pull/252
- P8 회고 §잘된것: https://github.com/coseo12/astro-simulator/blob/main/docs/retrospectives/p8-retrospective.md
- 선행 볼트 이슈: #21 (신규 함수 ≠ 신규 구현)
