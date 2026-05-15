---
title: Mode-gated mount precondition — UI 가드 도입 시 mount 전제조건과 가드의 직교성
type: knowledge
source_repo: coseo12/astro-simulator (downstream)
source_issue: 87
captured_at: 2026-05-09
status: refined
tags: [ui-guard, adr, design-axis, R-Phase, mount]
related: []
---

## 요약

UI 가드 도입 시 (i) panel/component 미렌더 vs (ii) render-but-disabled 선택 모호. mount precondition (panel 진입 가능 시점) 과 가드 (행위 차단 시점) 가 독립 축임을 인지 못 하면 ADR 후보 비교에서 누락. (ii) 채택 = mount precondition 독립 + UX 인지도 우선.

## 본문

**현상**: R-Phase 가드 같은 mode-gated UI 도입 시 두 축이 혼재한다.
- **mount precondition**: panel/component 가 언제 mount 되는가 (조건부 렌더 vs 항상 렌더)
- **guard**: mount 후 어떤 행위를 차단하는가 (클릭 무시 vs disabled UI)

두 축은 독립이지만 ADR 후보 비교에서 단일 축으로 뭉뚱그리면 (ii) render-but-disabled 의 UX 이점 (사용자가 항목 존재 인지 + 회색 disabled 로 미진입 안내) 이 누락된다.

**구체 사례**: PR #429/#430 (#403 CelestialTree + InfoPanel) — ADR 후보 비교 §축 2 에서 두 패턴 명시. (ii) 채택 = mount precondition 독립 + 사용자가 R-Phase 진입 전에도 panel 항목을 보고 학습 가능.

**재발 방지**: ADR 후보 비교 템플릿에 "mode-gated mount precondition 축" 신규 추가. UI 가드 ADR 은 항상 두 축을 명시.

## 관련 노트/링크

- astro-simulator PR #429/#430 (#403 CelestialTree + InfoPanel)
- docs/decisions/20260506-403-r-phase-ui-guard.md §축 2
