---
title: 라벨 일원화 계층 미전파 — skills 만 낙오한 status:* 잔존과 전파 체크리스트
type: knowledge
source_repo: coseo12/astro-simulator (이슈 #854, PR #860) / coseo12/harness-setting#320
source_issue: 124
captured_at: 2026-07-19
status: refined
tags: [label-taxonomy, skills-drift, layered-propagation, stage-labels]
related:
  - ./comment-contract-implementation-drift.md
  - ./skill-cli-snippet-empirical-check.md
  - ./ai-env-audit-7issue-z-pattern-cycle.md
---

## 요약

라벨 체계 일원화(status:*→stage:*, harness #127)가 **agents/commands 계층에는 이행됐지만 skills 계층에는 미전파**된 채 수개월 방치 — 의무 스킬(create-issue/create-pr)이 폐기 라벨을 계속 지시해 실제 PR 에 이중 라벨 오염이 발생했다. 교훈: 규약 일원화는 "참조하는 모든 계층"의 전수 grep 이 완료 조건이며, 라벨 명칭 divergence 는 문서 수정보다 **라벨 복원이 저비용**인 경우가 많다.

## 본문

**실측 (2026-07-18 전수 감사)**:
- create-issue 스킬 라벨 가이드 전면이 가상 라벨 (`agent:*` 13종/`status:todo~blocked`/`size:*`/`scope:*`/`type:feature`) — 실사용 시 gh 라벨 에러
- create-pr 의 `status:review` 부착이 PR #839 에 `status:review`+`stage:review` 이중 부착 실오염
- `stage:review` 를 **붙이는 주체가 15개 파일 어디에도 없음** — 전이 사슬 단절 (create-pr 이 죽은 라벨만 부착)
- `stage:done` 은 upstream 에 실존하는데 다운스트림 GitHub 만 `stage:ready-merge` 로 rename 된 divergence — 문서 5곳 수정(옵션 B) 대신 **라벨 복원(옵션 A)** 채택: open 부착 0건 실측이라 마이그레이션 비용 0, upstream 정합 + 파일 수정 0

**일원화 전파 체크리스트 (재사용)**:
1. 신·구 명칭 전수 grep — agents / commands / **skills** / scripts / workflows / 템플릿 전 계층
2. 전이 사슬 완결성 — 각 라벨을 "붙이는 주체"와 "떼는 주체"가 모두 문서에 존재하는지
3. GitHub 실물 라벨 셋과 문서 지시의 양방향 대조 (`gh label list` 실측 의무 — 스킬 템플릿 신뢰 금지)
4. legacy 라벨 삭제로 오염 재발 토양 제거 (closed 이력 라벨 소실은 허용 판단)

## 관련 노트/링크

- https://github.com/coseo12/harness-setting/pull/320 (upstream 해소 PR)
- https://github.com/coseo12/astro-simulator/pull/860
