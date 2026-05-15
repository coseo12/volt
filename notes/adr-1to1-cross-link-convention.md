---
title: ADR 1:1 박제 컨벤션 — 가드 1건 = ADR 1건 + cross-link 체인
type: knowledge
source_repo: coseo12/astro-simulator (downstream)
source_issue: 90
captured_at: 2026-05-09
status: refined
tags: [adr, architect, defense-in-depth, cross-link, governance]
related: []
---

## 요약

defense-in-depth 시리즈에서 정착한 컨벤션. 가드 1건당 ADR 1건 박제 + 부모/자매 ADR cross-link 체인 의무. 4건 ADR (#402 부모 → #415 url-sync → #403 UI → #404 presets) 이 일관 박제되어 변경 이력이 ADR 그래프로 관찰 가능.

## 본문

**현상**: 한 ADR 안에 여러 가드를 묶으면 (a) 후속 변경 시 어느 가드가 영향받는지 불분명 (b) cross-link 의 단위가 깨짐 (c) PR-ADR 1:1 매핑이 불가능해 squash merge 시 추적 어려움.

**1:1 박제 시 이점**:
- ADR 그래프로 가드 의존 체인 시각화 (#402 ← #415 ← #403 ← #404)
- 각 PR 이 정확히 1 ADR 을 박제 → squash 후에도 ADR 파일이 SSoT
- 후속 Amendment 가 정확한 ADR 만 mutate (다른 가드에 영향 0)

**구체 사례**: defense-in-depth 시리즈 4건 ADR 체인
- \`20260504-r-phase-allowlist-guard.md\` (부모, #402)
- \`20260504-415-url-sync-guard.md\` (#415)
- \`20260506-403-r-phase-ui-guard.md\` (#403)
- \`20260507-404-scenario-presets-r-phase-guard.md\` (#404)

각 ADR §관련 ADR 섹션이 부모/자매 cross-link 박제.

**재발 방지**: architect.md "1 가드 = 1 ADR + cross-link" 정책 + 신규 ADR 템플릿에 §관련 ADR 섹션 의무화.

## 관련 노트/링크

- astro-simulator PR #428/#429/#431 (defense-in-depth ADR 시리즈)
- docs/decisions/2026050{4,6,7}-*-r-phase-*.md
