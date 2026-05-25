---
title: macOS 시스템 bash 3.2 associative array 미지원 — verify-*.sh 호환성 함정
type: knowledge
source_repo: coseo12/astro-simulator
source_issue: 95
captured_at: 2026-05-17
status: refined
tags: [bash, macOS, shell-compat, verify-script, ssot-guard, ci-portability]
related:
  - ./bash-script-isolation-dynamic-test.md
  - ./gh-cli-spawn-stdin-shell-metachar.md
---

## 요약

`#!/usr/bin/env bash` 만 박으면 macOS 에서 시스템 `/bin/bash` (GNU bash 3.2.57) 가 잡혀 `declare -A` (associative array) 가 `invalid option` 으로 실패한다. CI 의 Ubuntu runner 는 bash 4+ 라 통과하지만 로컬 macOS 개발자가 즉시 차단당한다. `verify-*.sh` 류 SSoT 가드 스크립트는 처음부터 bash 3.2 호환 패턴으로 작성 — parallel index array + `sort -u` 패턴이 단일 모범.

## 본문

**증상**:
```
scripts/verify-create-pr-ssot.sh: line 32: declare: -A: invalid option
declare: usage: declare [-afFirtx] [-p] [name[=value] ...]
exit: 2
```

**환경 진단**:
```bash
$ which bash; bash --version | head -1
/bin/bash
GNU bash, version 3.2.57(1)-release (arm64-apple-darwin25)
```

macOS 는 Apple 의 GPLv3 회피 정책으로 bash 3.2 (2007년 출시) 가 시스템 기본. `brew install bash` 로 4+ 설치 가능하나 `/usr/local/bin/bash` 또는 `/opt/homebrew/bin/bash` 로 분리되어 shebang `#!/usr/bin/env bash` 가 `/bin/bash` 를 잡으면 무용지물. 사용자 PATH 우선순위에 의존하는 휴리스틱.

**bash 3.2 미지원 기능 (verify-*.sh 작성 시 자주 쓰이는 것)**:
- `declare -A` / 모든 associative array (`map[key]=val`) — bash 4.0+
- `${parameter,,}` / `${parameter^^}` 대소문자 변환 — bash 4.0+
- `mapfile` / `readarray` — bash 4.0+
- `coproc` — bash 4.0+
- `&>>` redirect append — bash 4.0+

**호환 패턴 (verify-create-pr-ssot.sh 실측)**:

associative array (bash 4+) — **미지원**:
```bash
declare -A captured_content
for agent in "${AGENTS[@]}"; do
  captured_content[$agent]=$(grep ...)
done
```

parallel index array (bash 3.2 호환) — **권장**:
```bash
captured_content=()  # AGENTS[i] 와 1:1 대응 index
for agent in "${AGENTS[@]}"; do
  captured_content+=("$(grep ...)")
done
# 동일성 검증: sort -u 패턴
unique_count=$(printf '%s\n' "${captured_content[@]}" | sort -u | wc -l | tr -d ' ')
```

**예방**:
- macOS 개발자: 로컬 실측 의무 (CI 만 믿으면 사용자 첫 실행에서 차단)
- 신규 verify-*.sh 작성 시 첫 라인 박제: `# 호환성 메모: macOS 시스템 bash 3.2 호환 위해 associative array 미사용 — parallel index array + sort -u 로 동일성 검증.`
- 기존 verify-agent-ssot.sh (이미 호환 작성됨) 패턴 일관성 유지

## 관련 노트/링크

- 사례: astro-simulator PR #496 (#480 5 페르소나 .md create-pr SSoT drift 자동 가드) 의 첫 실행 차단
- 호환 작성된 선례: `scripts/verify-agent-ssot.sh` (harness#145)
- GNU bash 4.0 release note (associative array 도입): 2009-02-20
- Apple bash 3.2 freeze 배경: GPLv3 회피
