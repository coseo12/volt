---
title: agent-browser eval 검증 한계 — console.* spy + page.evaluate 의 false positive
type: knowledge
source_repo: coseo12/astro-simulator (downstream)
source_issue: 88
captured_at: 2026-05-09
status: refined
tags: [agent-browser, headless, qa, console-spy, eval]
related:
  - ./reviewer-qa-gate-skip-headless-self-pass.md
  - ./skill-cli-snippet-empirical-check.md
---

## 요약

agent-browser headless + \`page.evaluate(() => state)\` + \`console.warn\` spy 검증이 실 Chrome (D-T2) UX 와 불일치. spy 는 출력을 잡지만 실 dev console 표기 모양 (포매팅, source map, stack trace) 은 다를 수 있다. volt #78 (헤드리스 ≠ 실 브라우저) 의 eval 검증 변형.

## 본문

**현상**: QA 단계에서 agent-browser headless 환경의 \`console.warn\` spy 와 \`page.evaluate\` 로 R-Phase 가드 동작을 검증할 수 있지만, 실 Chrome 의 console UX (warning level 표기, source location, group expansion) 는 다르다. spy 가 호출 fact 만 잡으면 사용자가 실제로 보는 메시지의 가독성/위치는 검증 공백.

**구체 사례**: PR #421 (#415 url-sync 가드) — agent-browser 7-cell 매트릭스 PASS, 사용자 D-T2 실 Chrome 재확인 후 머지. spy 검증만으로 마무리했다면 console UX 회귀 잠복 가능.

**재발 방지**: qa.md 에 "console.* spy 검증 후 사용자 D-T2 실 Chrome console 재확인" 강제. agent-browser eval 결과는 "참고 신호", 최종 게이트는 실 브라우저. 시각 효과 / console UX 포함 검증은 volt #78 의 확장 적용.

## 관련 노트/링크

- volt #78 (headless 브라우저 검증 ≠ 실 브라우저)
- astro-simulator PR #421 (#415 url-sync 가드)
- harness-setting .claude/agents/qa.md
