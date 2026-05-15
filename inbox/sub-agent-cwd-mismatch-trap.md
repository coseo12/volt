---
title: Sub-agent cwd 분리 함정 — 메인/sub-agent cwd 불일치로 빌드/검증 오작동
type: knowledge
source_repo: coseo12/astro-simulator (downstream)
source_issue: 85
captured_at: 2026-05-09
status: inbox
tags: [sub-agent, cwd, monorepo, build, pnpm]
related: []
---

## 요약

메인 오케스트레이터 cwd 와 sub-agent cwd 가 다를 때 (worktree 진입, 패키지 cwd 변경 등) 상대 경로 기반 빌드/테스트 명령이 의도와 다르게 해석되는 함정. \`pnpm test apps/web\` 가 sub-agent cwd \`apps/web\` 에서 \`apps/web/apps/web\` 으로 해석되는 사례.

## 본문

**현상**: sub-agent 가 worktree 또는 모노레포 내 특정 패키지로 cwd 변경 후 작업하면 메인 오케스트레이터와 cwd 가 달라진다. sub-agent 가 반환한 빌드/테스트 명령이 상대 경로 기반이면 메인 컨텍스트에서 재실행 시 다른 경로로 해석된다.

**구체 사례**: PR #421 (#415 url-sync 가드) — \`pnpm test apps/web\` 명령이 sub-agent cwd 가 이미 \`apps/web\` 인 상태에서 발행되어 \`apps/web/apps/web\` 으로 해석. 빈 결과 → 결정적 PASS 오판 위험.

**재발 방지**: sub-agent 프롬프트 첫 줄에 \`pwd\` 출력 의무화. 반환 JSON 의 모든 빌드/테스트/검증 명령은 **절대경로 또는 워크스페이스 root 기준** 으로 표기. developer.md 가드 추가.

## 관련 노트/링크

- astro-simulator PR #421 (#415 url-sync 가드)
- harness-setting .claude/agents/developer.md
