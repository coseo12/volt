---
title: sub-agent auto-close 박제 누락 — closingIssuesReferences GitHub linkage 실측 누락 (volt #24 새 변형)
type: pattern
source_repo: coseo12/astro-simulator
source_issue: 93
captured_at: 2026-05-13
status: refined
tags: [sub-agent, 박제 검증, GitHub linkage, auto-close, volt-24]
related:
  - ./sub-agent-finalization-miss-pattern.md
  - ../../notes/stale-dev-server-port-collision.md
  - ./sub-agent-bg-process-leak-pattern.md
  - ./agent-browser-chrome-zombie-pattern.md
---

## 배경/상황

PR #457 (astro-simulator) qa sub-agent 디스패치 + qa PASS 보고 직후 메인 카나리아에서 발견. sub-agent 마무리 체크리스트 JSON `non_blocking_suggestions` 에 `"PR closing keyword 단일 'Closes #400' 형식 정확. 머지 시 #400 자동 close 예상"` 으로 보고. 메인이 머지 전 `gh pr view 457 --json closingIssuesReferences` 실측 → 빈 배열 `[]` 발견. 머지 후 #400 OPEN 잔존 재현 → 메인 `gh issue close 400 --comment "..."` 수동 close.

## 내용

- **sub-agent self-report 절차**: PR 본문 grep 으로 "Closes" / "Fixes" / "Resolves" 키워드 + colon 함정 (`Closes: #A, #B`) 확인. **본문 문법만 검증**.
- **누락된 검증**: GitHub 가 실제 linkage 를 인식하는지 `closingIssuesReferences` GraphQL 필드 실측.
- **본 사례 조건**: 문법 정확 (`Closes #400` 단독 줄, colon 없음, 본문 끝부분 위치) + 본문에 다른 reference 없음 + base=develop. 그럼에도 GitHub linkage 빈 배열.
- **머지 후 결과**: PR #457 squash merge `857761c` → #400 state=OPEN, closedAt=null. 메인이 `gh issue close 400 --comment "Closed by PR #457 squash merge (commit 857761c). GitHub auto-close 가 발화하지 않아 메인 오케스트레이터가 수동 close — closingIssuesReferences 빈 배열 우려 적중 (volt #24 sub-agent 박제 검증 누락 패턴)."` 로 수동 close.
- **volt #24 sub-agent 박제 누락 패턴의 GitHub linkage 차원 새 변형**. 기존 변형: PR 코멘트 박제 누락 / 라벨 전이 누락 / 백그라운드 프로세스 정리 누락 (#46, #52) / 좀비 정리 누락 (#79). 본 변형은 **자동화 시스템 (GitHub) 의 인식 여부** 차원.
- **가설**:
  - PR 본문이 매우 길면 (본 사례 ~8KB) GitHub parser 가 closing keyword 를 indexing 안 했을 가능성 (확인 필요)
  - base 가 default branch 가 아닐 때 (develop) `closingIssuesReferences` 는 등록되지만 실제 close 는 main 머지 시점에 발화 — 이 경우 빈 배열 자체가 비정상
  - PR 생성 직후 GitHub indexing 지연 (eventual consistency) 가능성 — 그러나 본 사례는 PR 생성 후 1시간+ 경과
- **재현 가능성**: 본 PR 1회 관찰. 동일 저장소 (astro-simulator) 의 다른 머지 PR 들 (#451, #443, #442, #441, #437) 의 `closingIssuesReferences` 회고 실측 시 패턴 식별 가능.

## 교훈 / 다음에 적용할 점

- **모든 외부 가시성 박제 sub-agent** (developer / qa / reviewer / pm / architect) 의 **마무리 체크리스트 JSON `auto_close_issue_states` 필드 검증** 시 본문 grep 만으로 부족. **`gh pr view <PR> --json closingIssuesReferences` 실측 의무** 추가.
- **SSoT 박제 위치 후보**:
  - `.claude/agents/*.md` 의 `## 마무리 체크리스트 JSON 반환` 섹션 — sub-agent 가 반환 전 실측 의무
  - harness real-lessons 의 `### sub-agent 검증 완료 ≠ GitHub 박제 완료` 블록 — 메인 오케스트레이터 검증 게이트
- **메인 오케스트레이터 책임**:
  1. **머지 전** sub-agent 반환 직후 `closingIssuesReferences` 1회 실측 (canary)
  2. **머지 직후** issue state 1회 실측 (post-merge verification)
  3. 빈 배열 또는 OPEN 잔존 발견 시 옵션: (A) PR 본문 갱신 (close keyword 줄 재저장 → GitHub re-index 유도) / (B) 머지 후 수동 close + 코멘트 박제
- **본 사례는 (B) 선택** — 머지 진행 후 수동 close + 코멘트로 SSoT 박제
- **volt #24 누적 회차**: PR #457 머지에서 **19회차** 발화 (astro-simulator MEMORY 기준)
