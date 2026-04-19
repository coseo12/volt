---
title: workflow_dispatch 트리거 2단계 함정 — default branch 제약 + PR 자동생성 권한 기본 OFF
type: report
source_repo: coseo12/astro-simulator
source_issue: 45
captured_at: 2026-04-19
status: refined
tags: [github-actions, workflow_dispatch, pull-request, permissions, gitflow, release]
related: []
---

## 배경/상황

astro-simulator v0.7.x 시점. `bench:baseline-remeasure` workflow 를 PR #238 로 도입하고 develop 에 머지. 이후 `gh workflow run bench-baseline-remeasure.yml --ref develop ...` 로 트리거 시도 → 실패:

```
HTTP 404: workflow bench-baseline-remeasure.yml not found on the default branch
```

GitHub Actions 의 `workflow_dispatch` 는 **default branch (main) 에 해당 파일이 존재해야** UI/CLI 에서 보인다. develop 에만 있으면 discover 불가.

정식 gitflow 경로로 v0.7.1 PATCH 릴리스 (`develop → main` merge commit release PR) 후 main 에 workflow 파일 반영되자 dispatch 성공. 이번엔 aggregate job 의 `peter-evans/create-pull-request@v7` 가 실패:

```
##[error]GitHub Actions is not permitted to create or approve pull requests.
```

저장소 Settings → Actions → General → "Allow GitHub Actions to create and approve pull requests" 체크박스 기본 OFF. branch push 자체는 성공했으므로 수동 `gh pr create` 로 우회 가능.

두 번째 실행부터 자동화하기 위해 `gh api -X PUT /repos/{owner}/{repo}/actions/permissions/workflow -F can_approve_pull_request_reviews=true` 로 설정 변경. 다음 dispatch 에서 PR 자동 생성 성공.

## 내용

**함정 1 — workflow_dispatch default branch 종속**:
- GitHub UI 의 "Run workflow" 버튼 + `gh workflow run` 둘 다 default branch 의 파일 목록을 기준으로 workflow 를 찾는다
- `--ref <branch>` 로 다른 브랜치의 workflow 파일을 실행시킬 수는 있으나, **파일 자체는 default branch 에 존재해야** 한다
- 결과: "설계 PR 머지 → 즉시 실행" 흐름이 기본 gitflow 에서 불가. release 까지 가야 실행 가능

**함정 2 — `can_approve_pull_request_reviews` 기본 false**:
- 저장소 기본값: `{"default_workflow_permissions":"read","can_approve_pull_request_reviews":false}`
- workflow 가 `permissions: pull-requests: write` 를 선언해도 이 저장소 레벨 플래그가 false 면 **PR 생성 API 자체가 거부**
- 조치: `gh api -X PUT /repos/{owner}/{repo}/actions/permissions/workflow -f default_workflow_permissions=read -F can_approve_pull_request_reviews=true`
- 변경 후 즉시 효과 (재시작 불필요)

**실측 증거**:
- 첫 실행 (Settings 변경 전): https://github.com/coseo12/astro-simulator/actions/runs/24621714905 — aggregate job failure
- 두 번째 실행 (Settings 변경 후): https://github.com/coseo12/astro-simulator/actions/runs/24624988691 — 전체 PASS, PR #249 자동 생성

## 교훈 / 다음에 적용할 점

- **workflow_dispatch 도입 PR 의 DoD 에 "default branch 반영 후 실행 검증" 명시** — 설계 PR 만 머지하고 DoD 체크박스 "실행 검증" 을 못 채우는 함정 방지
- **PR 자동 생성 workflow 는 Settings 선행 조건 확인을 README/workflow 파일 상단 주석에 박제**:
  ```yaml
  # 사전 조건: Settings → Actions → "Allow GitHub Actions to create and approve pull requests" ON
  # 또는: gh api -X PUT /repos/{OWNER}/{REPO}/actions/permissions/workflow -F can_approve_pull_request_reviews=true
  ```
- **2단계 릴리스 리듬**: workflow 도입 → 릴리스(main 반영) → 설정 변경 → 실제 실행. 각 단계의 의존성을 ADR 또는 PR 본문에 박제하면 후임자가 역추적 가능
- **가시성 개선**: 저장소 표준 체크리스트에 `can_approve_pull_request_reviews` 항목 추가 (레포 초기화 도구가 있다면 기본 ON 으로 세팅하는 스크립트 고려)
