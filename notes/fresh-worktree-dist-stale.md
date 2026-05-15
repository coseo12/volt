---
title: Fresh worktree dist stale — monorepo 신규 worktree 에서도 #70 재현
type: knowledge
source_repo: coseo12/astro-simulator (downstream)
source_issue: 86
captured_at: 2026-05-09
status: refined
tags: [worktree, monorepo, dist, stale, pnpm]
related:
  - ./monorepo-core-dist-stale.md
---

## 요약

volt #70 (monorepo dist stale) 의 fresh worktree 변형. 신규 worktree 진입 후 \`pnpm install\` 만 하고 \`pnpm -r build\` 또는 dist clean 없이 dev/test 로 진입하면 core 패키지 stale dist 를 dev 서버가 참조해 신규 변경 미반영. QA 가 결정적 PASS 로 오판 위험.

## 본문

**현상**: monorepo + pnpm workspace 환경에서 신규 worktree 진입 시, install 만으로는 워크스페이스 패키지 dist 가 빌드되지 않는다. core 패키지의 신규 변경이 \`dist/\` 에 반영되지 않은 상태에서 \`apps/*\` dev 서버가 stale dist 를 import → 변경 효과 0.

**구체 사례**: PR #432 (#404 ScenarioPresets) developer worktree — core 패키지 stale dist 참조로 신규 R-Phase 가드 미반영. QA 가 캐시된 빌드 위에서 결정적으로 동일 결과를 내며 "수정 효과 없음" 오판 가능.

**재발 방지**:
- worktree 진입 시 \`pnpm -r build\` 선행 의무화 (또는 watch 병행)
- \`scripts/setup-worktree.sh\` 스크립트화
- QA 에이전트 §선행 조건에 "fresh worktree → core build 선행 확인" 추가
- volt #70 의 fresh worktree 사례로 CLAUDE.md real-lessons block 보강

**Builds on**: volt #70 (monorepo dist stale)

## 관련 노트/링크

- volt #70
- astro-simulator PR #432 (#404 ScenarioPresets)
- harness-setting .claude/agents/qa.md
