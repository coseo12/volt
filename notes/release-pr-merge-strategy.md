---
title: Release PR merge 전략 — squash 의 drift 강제 vs merge commit + fast-forward 4단계
type: knowledge
source_repo: coseo12/harness-setting (v2.13.0 관찰 → v2.14.0 전환 → v2.15.0 박제)
source_issue: 37
captured_at: 2026-04-19
status: refined
tags: [gitflow, release, merge-commit, squash, fast-forward, adr]
related:
  - ./manifest-state-recovery-pattern.md
  - ../meta/patterns/dual-pr-drift-timeline-pattern.md
  - ./gitflow-drift-4tier-classifier.md
  - ./develop-dual-role-integration-staging.md
---

## 요약

gitflow 에서 `develop → main` release PR 을 **squash merge 로 머지하면** main 에 새 squash commit 이 생겨 develop 과 diverge → 매 릴리스마다 merge-back PR 강제 (release 1회 = 3 PR). **merge commit + fast-forward push** 로 전환하면 main tip 이 develop tip 을 직계 조상으로 포함 → merge-back 불필요.

## 본문

#### squash merge 기전 (문제)

```
main 이전:  [A]
develop:    [A] - [B feature squash]

release PR (develop → main, squash 머지) 후:
main:       [A] - [B' squash commit on main]   ← B' 는 develop 에 없음
develop:    [A] - [B feature squash]           ← B 는 main 에 없음

결과: main..develop = 0 (develop 은 main 의 조상이 없음)
      develop..main = 1 (B' 는 develop 에 없음)
→ drift warn, merge-back PR 필요
```

#### merge commit 기전 (해결)

```
release PR (develop → main, `gh pr merge <PR> --merge`) 후:
main:       [A] - [M merge commit]
                  ├── [A] (이전 main)
                  └── [B] (develop tip, 직계 조상)
develop:    [A] - [B]                          ← 이전 상태 유지

→ develop 이 main 의 조상 (git merge-base --is-ancestor 참)
→ `git push origin main:develop` fast-forward 가능 (force 아님)
```

#### 4단계 릴리스 워크플로

```
1. gh pr merge <release-PR> --merge    # merge commit 으로 머지
2. git push origin main:develop        # fast-forward (force 아님)
3. git tag vX.Y.Z + git push origin vX.Y.Z
4. gh release create vX.Y.Z ...
```

#### 대안 비교

| 대안 | 장점 | 단점 |
|---|---|---|
| A. merge commit + fast-forward (채택) | merge-back PR 제거. 구조적 해결 | main log 가독성 저하 (feature 커밋 섞임) |
| B. Action 자동 merge-back | squash 유지 + 자동화 | Action 구축/유지 비용 |
| C. 수동 merge-back (이전) | 변경 없음 | 매 릴리스 +1 PR (sync-only) |

## 교훈

1. **기본 merge 전략을 PR 타입별로 분리** — repo 기본값은 `--squash` (feature PR) 이지만 release PR 만 `--merge` 사용. GitHub 레포 기본 설정은 하나만 지정 가능하므로 PR template + reviewer 체크 필요
2. **fast-forward push 는 force-push 가 아님** — main 이 develop 의 후손이면 `git push origin main:develop` 는 정상 push. CRITICAL (파괴적 작업 경고) 적용 대상 아님
3. **release 후 sync 상태 선제 검증** — `harness doctor` 또는 동등 체크로 "main..develop = 0 && develop..main = 0" 확인하는 루틴 의무화

## 관련 노트/링크

- harness-setting ADR: docs/decisions/20260419-release-merge-strategy.md
- harness 릴리스 v2.14.0, v2.15.0
- volt #27 (매니페스트 원자성) 과 결이 다르나 "상태 기록 원자성" 공통 원리
