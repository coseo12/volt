---
title: Sub-agent worktree drift 우회 — architect ADR 격리 시 develop tip 동기화
type: knowledge
source_repo: coseo12/astro-simulator (downstream)
source_issue: 84
captured_at: 2026-05-09
status: inbox
tags: [sub-agent, worktree, architect, drift, gitflow]
related: []
---

## 요약

architect sub-agent 가 worktree 격리로 ADR 을 작성하는 동안 develop tip 이 격리 시점에 고정되어, 후속 developer/reviewer/qa 단계에서 메인 working tree 와 다른 base 로 검증되는 패턴. 마무리 이탈(volt #82)의 발현 시점 변형 — ADR 박제 중 base 가 진전했을 때.

## 본문

**현상**: architect sub-agent 는 worktree 로 격리되어 ADR 본문을 박제한다. 박제 동안 메인 develop tip 은 다른 PR 머지로 진전 가능. architect 반환 후 developer/reviewer/qa 가 worktree 의 ADR 만 보고 진입하면 격리 시점-현재 격차를 인지하지 못해 stale base 충돌이 잠복한다.

**구체 사례**: PR #422 (411-r1-guard-shortcut-bar-forensic ADR) — architect worktree 박제 동안 다른 PR 머지로 develop tip 진전. 후속 단계에서 rebase 누락 시 충돌.

**재발 방지**: architect 반환 직후 메인 오케스트레이터가 \`git fetch + git log origin/develop..HEAD\` 로 격리 시점-현재 격차를 확인하고, 격차 발견 시 developer 진입 전 rebase. CRITICAL #1 브랜치 보호와 직교하는 격차 가드.

**Builds on**: volt #82 (sub-agent 마무리 이탈 + worktree base drift)

## 관련 노트/링크

- volt #82
- astro-simulator PR #422
- harness-setting .claude/agents/architect.md
