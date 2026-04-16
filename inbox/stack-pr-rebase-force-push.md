---
title: Stack PR 중간 머지 후 상위 PR의 필수 rebase + force-push
type: report
source_repo: coseo12/astro-simulator
source_issue: 17
captured_at: 2026-04-16
status: inbox
tags: [github, pr-workflow, rebase, stack-pr]
related: []
---

## 배경/상황

P4 마일스톤에서 의존 관계가 있는 PR 4개를 생성:

- #168 (P4-B, base=main)
- #169 (P4-D, base=main)
- **#170 (P4-A, base=feature/p4-d-gpu-timer)** ← stack PR
- #171 (P4-C, base=main)

#170은 P4-D의 API를 사용하므로 P4-D 브랜치 위에 쌓였다. P4-D(#169) 머지 후 #170을 main에 머지하려 했는데 GitHub UI에서 `gh pr edit 170 --base main`만으로는 **mergeable=CONFLICTING** 상태가 됨.

## 내용

**원인**

- GitHub가 base 변경 시 자동 rebase를 하지 않음
- head 브랜치는 여전히 옛 base(P4-D 브랜치)의 커밋을 포함 → main에 이미 포함된 커밋이 중복 감지 + package.json 등 공통 파일에서 실제 conflict 발생

**해결 절차**

```bash
# 1. head 브랜치 체크아웃
git checkout feature/p4-a-asteroid-nbody

# 2. 최신 main 기준으로 rebase
git fetch origin
git rebase origin/main
# → "skipped previously applied commit" (main에 이미 머지된 P4-D 커밋)
# → 실제 conflict 발생 시 수동 해결 + git rebase --continue

# 3. force-push (안전한 flag 사용)
git push --force-with-lease origin feature/p4-a-asteroid-nbody

# 4. 그제서야 mergeable
gh pr merge 170 --squash
```

**관찰**

- `--force-with-lease`는 원격이 내가 본 커밋과 일치할 때만 push → 동시 작업 시 사고 방지
- conflict는 주로 `package.json` 스크립트 목록, `MEMORY.md`, `CHANGELOG.md` 같은 append-heavy 파일에서 발생
- P4 세션에서 #170, #171 둘 다 동일 rebase 과정 필요 (둘 다 `verify:all` 체인을 수정)

## 교훈 / 다음에 적용할 점

**agent/skill 개선 포인트**

1. **stack PR 생성 시 체크리스트에 "머지 순서 + rebase 필요" 명시** — `create-pr` 스킬이 base가 main이 아닐 때 경고 추가 가능
2. **자주 수정되는 파일(`package.json`, `CHANGELOG.md`)은 stack PR 간 충돌 다발 영역** — 같은 섹션을 여러 PR이 수정하면 하위 PR은 거의 확실히 conflict
3. **`gh pr edit --base main` 후 mergeStateStatus 확인** → DIRTY/CONFLICTING이면 자동으로 로컬 rebase 유도
4. **대안**: stack 대신 **독립 브랜치**로 분할 + 의존성은 기능 플래그/옵트인으로 해결. 본 세션의 P4-D → P4-A는 실제로는 P4-A를 P4-D 없이도 main 기반으로 만들 수 있었음 (P4-D API를 선택적 import)
