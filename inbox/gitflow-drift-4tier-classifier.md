---
title: gitflow drift 감지 4단 분류 — is-ancestor + hotfix 문맥 + unrelated histories 방어
type: knowledge
source_repo: coseo12/harness-setting (v2.15.0 `lib/doctor.js` `classifyGitflowDrift`)
source_issue: 38
captured_at: 2026-04-19
status: inbox
tags: [gitflow, doctor, drift-detection, merge-base, false-positive, false-negative]
related:
  - ../notes/state-atomicity-multi-layer-defense.md
---

## 요약

main vs develop 커밋 격차만 단순 count 하면 **거짓 양성 (release merge commit 직후 warn)** 과 **거짓 음성 (unrelated histories 에서 null → 0 수렴)** 이 발생. 4단 우선순위 분류로 해소:

```
null 입력 (rev-list 실패)      → warn (unrelated histories 의심)
  ↓
mainAhead === 0                 → pass (동기 또는 develop 앞섬)
  ↓
developIsAncestorOfMain         → pass (fast-forward 동기화 대기 — merge commit 직후 정상)
  ↓
hasHotfixBranch                 → warn (hotfix 진행 중, merge-back PR 필요)
  ↓
기타                            → warn (hotfix 누락 또는 release squash 실수)
```

## 본문

#### 각 분기 논거

1. **null 방어**: `git rev-list --count` 가 unrelated histories 에서 exit 1 → tryExec null → parseInt("0")=0 수렴으로 **거짓 음성**. 명시적 null 분기로 `git merge-base` 확인 안내
2. **ancestor pass**: release PR 을 `gh pr merge --merge` 로 머지하면 main 에 merge commit 생성. 이 commit 은 develop 에 없으므로 `mainAhead=1`. 하지만 `git merge-base --is-ancestor develop main` 는 참. 이 상태는 fast-forward push 한 번으로 해소되는 **정상 중간 상태**. 거짓 양성 제거 필수
3. **hotfix 문맥**: `hotfix/*` 브랜치 존재 시 main 이 의도적으로 develop 보다 앞설 수 있음. 단순 drift warn 대신 "hotfix 진행 중 — merge-back 필요" 로 세분화하면 사용자 혼란 감소
4. **기타 warn**: 실제 drift (hotfix merge-back 누락 또는 release 을 실수로 `--squash` 로 머지)

#### 순수 함수 + 테스트 분리

```js
function classifyGitflowDrift(mainAhead, devAhead, opts = {}) {
  const { developIsAncestorOfMain = false, hasHotfixBranch = null } = opts;
  // ... 4단 우선순위 분류
}
```

호출부에서 git 호출 후 결과만 전달 → 단위 테스트가 git 없이 가능. 테스트 28건 (기존 22 + drift 세분화 6).

## 교훈

1. **drift 감지 로직은 pure function 으로 분리** — 단위 테스트 커버리지 높이기
2. **Ancestor 체크가 핵심** — squash/merge-commit 구분에 `--is-ancestor` 가 가장 신뢰 가능한 지표
3. **상태 기록 원자성 3계층 방어 (volt #28) 연장** — 계층 3 (사용자 안내) 의 품질이 경계 케이스 분류로 결정됨. "warn" 은 구체적 복구 경로까지 포함해야 유용

## 관련 노트/링크

- harness-setting `lib/doctor.js` classifyGitflowDrift
- 교차검증 고유 발견: Gemini 2차 cross-validate (harness #107)
- volt #28 (상태 기록 3계층 방어) 의 확장
