---
title: ADR Amendment 위치 결정 트리 — 동일 이슈 → mutation, 신규 이슈 → 신규 ADR + cross-link
type: knowledge
source_repo: coseo12/astro-simulator (downstream)
source_issue: 91
captured_at: 2026-05-09
status: inbox
tags: [adr, amendment, governance, decision-tree, architect]
related: []
---

## 요약

ADR 후속 변경 시 (a) 원 ADR 에 Amendment 박제 (file mutation) vs (b) 신규 ADR 박제 (file add + 부모 ADR cross-link Amendment 동시) 선택의 결정 트리. 동일 이슈/라운드 → (a), 신규 이슈/라운드 → (b). volt #83 (Amendment 마크업 SSoT) 의 직교 축 — 어디에 둘지.

## 본문

**현상**: ADR 후속 변경 시 두 경로 중 선택 모호.
- **(a) file mutation**: 원 ADR 본문에 Amendment 섹션 추가 → 동일 이슈/라운드 내 정정/세부 조정에 적합
- **(b) file add + cross-link**: 신규 ADR 박제 + 부모 ADR 에 Amendment cross-link 동시 박제 → 신규 이슈/후속 라운드에 적합

**결정 트리**:
\`\`\`
이 변경은 어느 이슈/라운드에서 발생했는가?
├─ 동일 이슈 / 동일 라운드 (예: baseline 정정, 세부 조정)
│   → (a) 원 ADR 에 Amendment mutation
│
└─ 신규 이슈 / 후속 라운드 (예: 형제 가드 추가, 정책 확장)
    → (b) 신규 ADR + 부모 ADR cross-link Amendment 동시
\`\`\`

**구체 사례**:
- **(a) PR #426**: #362 R1 ADR (sun 점유율) — baseline 재캡처로 6.18% → 3.88% 정정 → 동일 이슈 라운드 내 → 원 ADR Amendment v2 mutation
- **(b) PR #432**: #404 ScenarioPresets — 신규 이슈 + 부모 #403 ADR §"R-Phase 진입 시 7 분기" 형제 가드 추가 → 신규 ADR + 부모 ADR Amendment 동시 박제

**재발 방지**: architect.md "Amendment scope 판정 트리" 명시 + ADR 템플릿에 "Amendment 위치" 결정 가이드 추가.

**Builds on**: volt #83 (ADR 정정 패턴 SSoT — 마크업 직교 축)

## 관련 노트/링크

- volt #83 (ADR 정정 패턴 SSoT)
- astro-simulator PR #426 (R1 ADR Amendment v2)
- astro-simulator PR #432 (#404 ADR + #403 부모 ADR Amendment 동시)
- docs/decisions/20260425-r1-ui-pixel-diff-guard.md Amendment v2/4
