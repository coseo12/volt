---
title: ADR-코드 분리 PR 패턴 trade-off — 보존/revert vs drift conflict
type: knowledge
source_repo: coseo12/astro-simulator
source_issue: 81
captured_at: 2026-05-04
status: refined
tags: [pr-pattern, adr, conflict-resolution, gitflow, sprint-pattern, monorepo]
related:
  - ./lint-staged-gitignore-silent-revert.md
  - ../meta/patterns/sub-agent-finalization-miss-pattern.md
---

## 요약

한 sprint 의 **ADR 박제와 코드 구현을 별도 PR 로 분리**하면 라운드 1 같은 회귀 시 코드 PR 만 close + revert 가능 (ADR 보존 이점). 단점: ADR 본문이 라운드 정정 commit 으로 갱신되면 코드 PR 의 동일 문서 본문이 drift → 머지 시 충돌 발생. **해소**: ADR PR 먼저 머지 → 코드 PR head 에 `git merge origin/develop` + `git checkout --theirs <ADR 파일>` 로 develop 정정본 채택 → 머지 commit push → squash 머지.

## 본문

#### 분리 패턴의 이점 (라운드 1 사례)

라운드 1 (#402 sprint) 에서 ADR PR (#406, merged 3c90844) 와 developer 코드 PR (#407) 분리.

- developer 코드 PR (#407) 가 turbopack `__dirname` SSR 500 회귀로 closed/revert
- **ADR PR (#406) 은 develop 에 보존** — 4축 결정 + defense-in-depth 4곳 동시 박제 절차 그대로 유지
- 라운드 2 architect 가 보존된 ADR 위에 §Amendment 추가만 하면 됨 (재작성 0)

#### 분리 패턴의 단점 (라운드 2 사례)

라운드 2 (#402 sprint) 에서 동일 분리:
- PR #413 (ADR Amendment, head=`fix/402-r-phase-allowlist-guard-round2`)
- PR #414 (코드 구현, head=`fix/402-r-phase-allowlist-guard-round2-impl`, base=develop)

라운드 2 정정 commit (`4e1670a`) 가 PR #413 에 push 되면서 ADR §결정 D1 본문이 namespace re-export 폐기 + named export 직접 박제 패턴으로 갱신. 그러나 PR #414 head 의 ADR 본문은 정정 전 (라운드 2 초안). 머지 순서 강제 (#413 → #414) 에서:

```
1. PR #413 머지 → develop ADR 갱신 (정정본)
2. PR #414 머지 시도 → ADR 파일 conflict (PR #414 의 초안 vs develop 의 정정본)
```

#### 해소 절차 (squash 머지 + theirs 채택)

```bash
# 코드 PR head 브랜치에 develop 의 정정본 통합
git checkout fix/402-r-phase-allowlist-guard-round2-impl
git fetch origin develop
git merge origin/develop --no-edit
# CONFLICT (content): docs/decisions/<ADR>.md

# develop 의 ADR 정정본 채택 (--theirs = origin/develop)
git checkout --theirs docs/decisions/<ADR>.md
git add docs/decisions/<ADR>.md
git commit --no-edit  # 머지 commit
git push origin <head>

# 다시 squash 머지 시도
gh pr merge <코드PR> --squash --delete-branch
```

→ **squash 머지** 이므로 머지 commit 자체는 develop history 에 남지 않고 단일 커밋으로 통합. ADR 본문은 develop 의 정정본 그대로 유지.

#### 일반화

> **ADR/문서 PR 이 코드 PR 보다 먼저 머지될 때 코드 PR 의 동일 문서 본문은 develop 으로 따라가야 한다.**

해당 패턴이 적용되는 모든 분리 PR 시나리오 (ADR + 코드 / 정책 문서 + 구현 / 스키마 + 마이그레이션 등) 에 일반화 가능. 머지 순서 강제 + 충돌 시 `--theirs` 채택은 reviewer 가 **머지 순서 권고** 로 박제할 의무.

#### 분리 vs 단일 PR 결정 기준

| 시나리오 | 권장 |
|---|---|
| 라운드 1 회귀 가능성 (실험적 구현, 차단 위험 높음) | **분리** — ADR 보존 가치 > drift 비용 |
| 안정적 구현 (재구현 위험 낮음) | **단일** — drift 회피 |
| ADR 단독 가치 (구현 없이도 정책 박제 의의) | **분리** — ADR 즉시 반영 가능 |
| 머지 순서 강제 비용이 큰 경우 (다수 의존 PR) | **단일** — 단순화 |

#### 라벨/단계 게이트 영향

분리 시 reviewer 는 두 PR 동시 검토 (volt #77 단계 게이트 유지). qa 는 코드 PR 만. ADR PR 의 라벨 전이는 reviewer 권고 없으면 자체 흐름.

## 관련 노트/링크

- 라운드 1 ADR 보존 PR (merged 3c90844): https://github.com/coseo12/astro-simulator/pull/406
- 라운드 1 코드 PR (closed 3eed4e0): https://github.com/coseo12/astro-simulator/pull/407
- 라운드 2 ADR Amendment (merged 4975846): https://github.com/coseo12/astro-simulator/pull/413
- 라운드 2 코드 PR (merged d2f6adf): https://github.com/coseo12/astro-simulator/pull/414
- conflict 해소 commit: e504c00
- 관련 volt 교훈: #13 (lint-staged silent partial commit), #24 (sub-agent → 메인 박제 인계 — auto-close 누락 패턴)
