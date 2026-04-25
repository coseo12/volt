---
title: lint-staged + .gitignore 상호작용으로 staged 변경이 조용히 유실되는 버그
type: report
source_repo: coseo12/astro-simulator
source_issue: 13
captured_at: 2026-04-15
status: refined
tags: [lint-staged, prettier, gitignore, husky, pre-commit, git-add, silent-revert]
related:
  - harness-frozen-file-split-pattern.md
  - ./manifest-state-recovery-pattern.md
  - ./state-atomicity-multi-layer-defense.md
  - ./harness-downstream-prettier-drift.md
  - ./stale-dev-server-port-collision.md
  - ./ci-green-no-test-execution-trap.md
  - ./external-tool-claim-empirical-verify.md
  - ./release-version-metadata-drift-guard.md
  - https://github.com/coseo12/astro-simulator/pull/163
  - ./monorepo-core-dist-stale.md
  - ./fact-mode-unification-ux-regression.md
---

## 배경/상황

harness-setting v2.3.0 → v2.4.0 업데이트 브랜치(`chore/harness-update-v2.4.0`)에서 다음 파일을 stage 후 커밋:

- CLAUDE.md (managed-block 동기화 + 신규 섹션)
- docs/* 17건 (템플릿 동기화)
- .github/ISSUE_TEMPLATE/task.yml, .github/PULL_REQUEST_TEMPLATE.md
- .github/workflows/ci.yml, ci-physics-wasm.yml(신규)
- .claude/skills/run-tests/SKILL.md (인덱스에 이미 tracked)
- .harness/manifest.json

`.gitignore` 에 `.claude/` 디렉토리 규칙이 있었지만 `.claude/skills/run-tests/SKILL.md` 는 과거부터 tracked 상태였음.

## 내용

### 증상

`git commit` 실행 시 lint-staged 가:

1. 원본 상태 stash 백업 ✓
2. 22 파일 prettier --write 성공 ✓
3. **"Applying modifications from tasks..." 단계에서 `.claude` 경로가 gitignore 된다는 오류로 FAILED**
4. 커밋은 **성공 메시지 출력**하며 종료 — 하지만 실제로는 4 파일만 반영:
   - `.claude/skills/run-tests/SKILL.md`, `ci.yml`, `ci-physics-wasm.yml`, `manifest.json`
5. **CLAUDE.md + docs/* 17건 + 템플릿 파일은 stage 됐음에도 working tree 까지 원본 상태로 되돌려짐** (stash 복원 과정에서 유실)

`git diff main HEAD -- CLAUDE.md` 실행 시 0 — 즉, 커밋에 안 들어간 상태로 working tree 까지 원본 복구됨.

### 원인

- `.gitignore` 에 `.claude/` 규칙이 존재하지만 일부 하위 파일이 **이미 인덱스에 tracked**
- lint-staged 의 modifications 재적용(`git add` 재실행) 단계에서 `.claude/` 경로에 add 시도 → gitignore 위배로 실패
- 실패 처리에서 stash 복원이 트리거되어 **다른 경로의 staged+working 변경까지 함께 롤백**됨
- 커밋 자체는 이미 pre-commit 훅 이전 stage 된 일부 내용으로 성공 → 사용자에게 실패가 명확히 전달되지 않음

### 재현 조건

1. `.gitignore` 에 디렉토리 규칙 추가
2. 해당 디렉토리 하위에 과거부터 tracked 된 파일이 혼재
3. lint-staged + prettier(혹은 유사한 "staged 파일 수정 후 재-stage" 훅) 구성

### 대응

- **즉시 복구**: 업스트림에서 파일 재-fetch 하거나, 유실된 변경을 재-stage 해서 재-commit
- **근본 해결**:
  - tracked ignored 파일을 `git rm --cached` 로 정리
  - 또는 `.gitignore` 에 예외 규칙 추가 (예: `!.claude/skills/run-tests/SKILL.md`)
- **예방**: lint-staged 출력에서 `[FAILED]` 라인을 발견하면 커밋 성공 메시지를 그대로 믿지 말고 `git show --stat HEAD` 로 실제 파일 목록 검증

## 교훈 / 다음에 적용할 점

1. **"커밋 성공 ≠ 의도한 변경이 커밋됨"** — lint-staged 가 내부적으로 일부 경로 apply 에 실패하면 staged 변경 일부가 조용히 사라질 수 있다. 빌드 성공 ≠ 동작, HTTP 200 ≠ 올바른 리소스 원칙의 연장선.

2. **harness 가드 제안**: 커밋 직후 `git diff <base> HEAD -- <예상 파일 목록>` 으로 기대 변경이 실제로 들어갔는지 검증하는 단계를 agent workflow / PR 체크에 추가. 한국어 파일 Edit 후 U+FFFD 검증과 같은 급의 "결과 재확인" 가드로 격상할 가치가 있음.

3. **lint-staged 출력 해석 규칙**: `[FAILED]` 키워드가 등장하면 **무조건 커밋 후 검증**. 특히 `gitignore` 관련 에러는 조용한 롤백의 전조.

4. **tracked + ignored 혼재 상태 금지**: 새 `.gitignore` 규칙 추가 시 해당 범위에 이미 tracked 된 파일이 있는지 `git ls-files <path>` 로 확인하고 `git rm --cached` 로 정리하는 체크리스트 항목이 harness 에 필요.

## 보강 (2026-04-15 후속, PR #163 커밋 `0af4c00`)

이슈 최초 캡처 후 동일 브랜치에서 추가 커밋(`chore(harness): 포맷 정리 + next-env 자동갱신`)을 시도하며 재현된 패턴.

### 우회책 검증: `git add -f`

`.claude/` 하위 harness 파일(`.claude/agents/*.md`, `.claude/skills/*.md`)을 재스테이지할 때 `git add -f` 를 사용하면:

- lint-staged 는 여전히 `[FAILED] The following paths are ignored by one of your .gitignore files: .claude` 메시지를 출력
- **그러나** 커밋 자체는 의도한 5개 파일 모두 반영됨 — 이번에는 조용한 롤백이 발생하지 않음
- 차이: 이번에는 `git add -f` 로 명시적으로 강제 스테이지했기 때문에 lint-staged 내부 재적용 단계가 실패해도 사용자가 원본 stage 한 상태가 보존됨 (스태시 복원 경로가 다르게 동작하는 것으로 추정)

즉 **`git add -f` 는 실전 우회책으로 유효**하지만, lint-staged 의 `[FAILED]` 출력은 여전히 혼란스러움 — "실패했다고 하는데 커밋은 성공했다"는 인지 부하가 남음.

### 근본 해결안 재정리 (배포 구조 관점)

기존 제안(`git rm --cached` / `!path` 단건 예외)에 더해 harness 배포 구조 차원의 2가지 옵션:

1. **`.gitignore` 화이트리스트 예외**: `!.claude/agents/`, `!.claude/skills/`, `!.claude/CLAUDE.md` 등 tracked 대상만 선별 허용.
   - 장점: 기존 구조 유지.
   - 단점: harness 파일 추가 시마다 예외 규칙 동기화 필요.
2. **harness 파일 `.harness/` 하위로 이동**: `.claude/` 는 완전히 무시 대상으로 두고, 공유 파일은 `.harness/agents/`, `.harness/skills/` 로 배치 후 Claude Code 측에서 심볼릭 링크 또는 로더 경로 수정.
   - 장점: ignore 규칙과 tracked 영역이 명확히 분리.
   - 단점: Claude Code 로더 경로 관습과 충돌 가능성.

### 교훈 보강

- `git add -f` 는 해결책이 아니라 **증상 억제책** — 근본 원인인 tracked+ignored 혼재 상태는 남음.
- harness-update 스킬 / PR 워크플로에 **"커밋 후 `git show --stat HEAD` 파일 목록이 기대와 일치하는지" 자동 검증** 단계를 넣는 것을 재차 제안 (이미 위 2번 항목 제안 → 운영 우선순위 격상).
