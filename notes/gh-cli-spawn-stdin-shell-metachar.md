---
title: gh CLI 마크다운 본문 발송 — execSync shell metachar 함정 + spawnSync stdin 우회 패턴
type: knowledge
source_repo: coseo12/astro-simulator
source_issue: 114
captured_at: 2026-05-25
status: refined
tags: [nodejs, gh-cli, shell-escape, execsync, spawnsync, stdin, markdown-body, comment-automation]
related:
  - ./macos-bash-3-2-associative-array.md
  - ./yaml-heredoc-indent-trap.md
---

## 요약

Node.js 에서 `execSync('gh pr comment N --body "..."')` 로 마크다운 본문 (백틱 / `$` / 특수 문자 포함) 발송 시 **shell metachar 가 명령 치환·변수 확장으로 해석**되어 syntax error 발생. `spawnSync('gh', [..., '--body-file', '-'], { input: body })` + stdin 으로 shell 우회 패턴이 일반 해결.

## 본문

**증상 (PR #497 D4 시뮬레이션 negative case)**:

```javascript
// 실패 코드
import { execSync } from 'node:child_process';

function postComment(prNumber, body) {
  execSync(\`gh pr comment ${prNumber} --body ${JSON.stringify(body)}\`, { stdio: 'inherit' });
}

// body 안에 마크다운 백틱이 포함:
//   "## PR 템플릿 체크리스트 가드 (#473)"
//   "phrase \\"\`보안\`\\""
```

GitHub Actions runner 로그:
```
/bin/sh: 1: Syntax error: end of file unexpected
코멘트 발송 실패: Command failed: gh pr comment 497 --body "...\`보안\`..."
```

**원인**:
1. `execSync(commandString)` 는 \`/bin/sh -c <commandString>\` 으로 실행
2. shell 이 본문의 \` (백틱) 을 **명령 치환** 으로 해석 → 미완성 백틱 짝 → syntax error
3. `JSON.stringify` 가 백슬래시 이스케이프 처리하지만 shell 은 JSON 이스케이프 무시 → shell parser 에 raw 로 도달

**해결 (spawnSync + stdin)**:

```javascript
import { spawnSync } from 'node:child_process';

function postComment(prNumber, body) {
  const result = spawnSync('gh', ['pr', 'comment', prNumber, '--body-file', '-'], {
    input: body,
    stdio: ['pipe', 'inherit', 'inherit'],
    encoding: 'utf-8',
  });
  if (result.status !== 0) {
    console.error(\`코멘트 발송 실패: exit ${result.status}\`);
  }
}
```

**3축 우회**:
1. `spawn` 계열 사용 — args 배열로 분리 (shell 미사용)
2. `--body-file -` — stdin 으로 본문 전달 (args 보다 안전, 긴 본문 가능)
3. \`{ input: body }\` — Node.js 가 child stdin 에 자동 pipe

**일반화 — Node.js 에서 외부 도구 호출**:

| 본문 특성 | execSync (shell) | spawnSync + args | spawnSync + stdin |
|-----------|------------------|-------------------|--------------------|
| 짧은 ASCII 만 | OK | OK | OK |
| 특수 문자 (`$ \` \\ ! ;`) | **위험** | OK | OK |
| 긴 본문 (> 1KB) | OS arg limit | 위험 (ARG_MAX) | **권장** |
| 마크다운 / JSON | **위험** | OK | **권장** |

**예방 (가드 SSoT 박제 후보)**:
- Node.js 에서 외부 CLI 호출 시 본문이 사용자 / template 생성이면 spawnSync + stdin 의무
- execSync 는 **고정 문자열 / 환경 변수 없는 명령** 에만 사용

## 관련 노트/링크

- 사례: astro-simulator PR #497 (#473 D4 CI backstop, run 25992144697 FAILURE → fix `a75aa20`, 2026-05-17)
- 관련 volt: #95 (macOS bash 3.2), #103 (YAML block scalar + bash heredoc indent)
- Node.js 공식: https://nodejs.org/api/child_process.html#child_processspawnsynccommand-args-options
