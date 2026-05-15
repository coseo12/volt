---
title: Lint/encoding 발견 PR 외 분류 — Fixes vs Refs 엄격 구분
type: knowledge
source_repo: coseo12/astro-simulator (downstream)
source_issue: 89
captured_at: 2026-05-09
status: inbox
tags: [pr-policy, lint, fixes, refs, auto-close]
related: []
---

## 요약

PR 검증 중 발견한 사전 lint 부채를 본 시리즈 PR 에 끼워 넣을지 분리할지의 정책. 본 이슈와 무관한 부채를 "Fixes #4XX" 로 묶으면 auto-close 오작동 + 리뷰 산만. encoding/formatting 만 본 PR DoD 로, 나머지 lint 부채는 신규 이슈 + Refs 로만 기록.

## 본문

**현상**: PR 작업 중 lint 부채 (turbopack SSR 회귀 / \`setState in effect\` / import() type 등) 가 자주 발견된다. 본 이슈 범위에 포함시키면:
- auto-close 오작동 (관련 없는 이슈가 닫힘)
- 리뷰 범위 확장 → 리뷰 지연
- 회귀 시 원인 탐색 어려움

**구체 사례**:
- PR #421 (#415 url-sync) 검증 중 발견된 turbopack \`__dirname\` SSR 회귀 → 별도 PR #413 분리
- PR #432 (#404 ScenarioPresets) 검증 중 lod-dev-overlay setState in effect → 신규 이슈 #386 분리

**재발 방지**: developer.md "lint PR 분리 체크리스트"
- encoding/formatting 만 본 PR DoD 로 (CRITICAL #4 한글 인코딩 검증 직결)
- 나머지 lint 부채 (SSR / hook 규칙 / 타입) 는 **신규 이슈 + Refs** 로만 기록
- "Fixes" 키워드는 본 이슈/직접 종결 대상에만 사용

## 관련 노트/링크

- astro-simulator 이슈 #386 (P10 lint 잔존)
- astro-simulator PR #413 (turbopack __dirname SSR 회귀)
