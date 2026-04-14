---
title: 워크스페이스 테스트 설정 누락으로 9 PR이 테스트 없이 머지됨
type: feedback
source_repo: coseo12/astro-simulator
source_issue: 2
captured_at: 2026-04-14
status: inbox
tags: [process, vitest, monorepo, pnpm-workspace]
related: []
---

## 배경/상황

pnpm 모노레포에서 apps/web 워크스페이스에 vitest 설정이 빠진 채 D 그룹(UI 레이어) 9개 PR이 머지되었다. 각 PR은 core 패키지의 기존 테스트만 실행했고 apps/web에는 단위 테스트가 1건도 없었음. 사용자 지적으로 fix PR(#68)에서 설정 추가.

## 내용

- 스캐폴딩 단계(A6)에서 모든 워크스페이스에 테스트 기반을 마련했어야 했으나 apps/web만 누락.
- `pnpm -r test`는 scripts.test가 없는 워크스페이스를 조용히 건너뜀.
- 9개 UI PR이 이를 "테스트 0건이지만 PASS"로 통과시킴.
- 수습: 루트에 `verify:test-coverage` 스크립트 추가 — 각 apps/*, packages/* 디렉토리에 vitest.config + scripts.test 존재를 검사, 누락 시 exit 1. CI 파이프라인과 `verify:all` 체인에 연결.

## 교훈 / 다음에 적용할 점

- **모노레포 스캐폴딩 시 "각 워크스페이스에 테스트 기반 필수" 체크리스트를 agent 템플릿에 박아두기.**
- `pnpm -r <script>`의 조용한 스킵 동작은 감시 장치가 없으면 사고로 이어짐. 루트에 "워크스페이스별 테스트 설정 존재 검사" 스크립트를 의무화.
- 신규 워크스페이스(P2의 physics-wasm 등) 추가 시 동일 검사가 자동으로 걸리도록 설계.
- skill: create-package / scaffold 계열이 있다면 테스트 설정을 기본 포함.
