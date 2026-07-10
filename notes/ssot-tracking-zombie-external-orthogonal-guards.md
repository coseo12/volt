---
title: SSoT 추적 단위 외부 좀비 — 가드 A/B/C 직교 박제 패턴 (incident #440)
type: knowledge
source_repo: coseo12/astro-simulator (downstream)
source_issue: 92
captured_at: 2026-05-10
status: refined
tags: [zombie-process, ssot-gap, defense-in-depth, dev-server, hook, ci-guard, sub-agent, port-binding]
related:
  - ./stale-dev-server-port-collision.md
  - ../meta/patterns/sub-agent-bg-process-leak-pattern.md
  - ../meta/patterns/agent-browser-chrome-zombie-pattern.md
  - ../meta/patterns/sub-agent-finalization-miss-pattern.md
  - ./scattered-constant-data-meta-ssot.md
  - ./session-dead-wait-guard-3layer.md
---

## 요약

CLAUDE.md SSoT (\`spawned_bg_pids\` / \`bg_process_handoff\`) 는 **"sub-agent 가 시작한 PID"** 만 추적한다. 이전 세션 / 사용자 직접 / 외부 도구가 spawn 한 좀비는 추적 단위 외 — sub-agent 가 \`spawned_bg_pids: []\` 보고해도 정의상 정합 PASS 면서 좀비 잔존. 좀비가 단순 점유가 아닌 응답(HTTP / port LISTEN) 을 하면 메인이 "ready" 오인 → 사용자 EADDRINUSE 충돌. 해소: 가드 A (메인 spawn 직전 lsof) + 가드 B (sub-agent 복귀 직후 카나리아) + 가드 C (세션 시작 hook) 를 시간축 직교 배치 + verify-zombie-check 회귀 가드 자동화로 박제 자체의 회귀까지 차단.

## 본문

## 현상 — 3중 gap

CLAUDE.md SSoT 정의상:
- \`spawned_bg_pids\`: 반환 전까지 sub-agent 가 시작해 살아있는 PID 배열
- \`bg_process_handoff\`: \`"main-cleanup"\` / \`"sub-agent-confirmed-done"\` / \`"none"\`

본 incident (astro-simulator 2026-05-10) 에서 다음 3중 gap 동시 발화:

1. **Gap 1 — sub-agent 추적 단위 외부**: 이전 세션 잔존 좀비 (PID 97333, ETIME 3h 17m) 가 본 세션 sub-agent 추적 외 → \`spawned_bg_pids: []\` 보고 정의상 정합 PASS, 좀비 잔존
2. **Gap 2 — 메인 검증 트리거 미발화**: \`bg_process_handoff="sub-agent-confirmed-done"\` 보고에서 메인이 ps/lsof 독립 검증 우회
3. **Gap 3 — 메인 자기 spawn 시점 lsof 미선행**: 메인이 새 dev 서버 \`run_in_background=true\` 로 spawn 시도 → EADDRINUSE 즉사 (\`head -50\` stdout 닫힘으로 메인 인지 못 함) → 좀비가 HTTP 307 응답 → 메인 "dev ready" 오인 → 사용자 D-T2 안내 → 사용자 \`pnpm dev\` 시도 시 \`ERR_PNPM_RECURSIVE_RUN_FIRST_FAIL\` → 사용자 보고

→ SSoT 정의상 모든 가드 PASS 인데 좀비 잔존 + 메인 오인 + 사용자 차단 동시 발생.

## 가드 A/B/C 직교 박제 패턴

3 가드를 시간축으로 직교 배치 — 단일 가드 누락 시 다음 가드가 백업:

| 가드 | 발동 시점 | 검증 대상 | 박제 위치 |
|------|----------|----------|----------|
| **A** | 메인이 새 dev/장기 프로세스 spawn 직전 | 본 세션 안 자기 spawn 좀비 | CLAUDE.md \`#### 가드 A — 메인 spawn 시점 lsof 선행\` |
| **B** | sub-agent 복귀 직후 (보고 무관) | sub-agent 추적 외부 + 본 세션 좀비 | CLAUDE.md \`#### 가드 B — sub-agent-confirmed-done 카나리아\` + qa.md 좀비 카나리아 |
| **C** | Claude Code 세션 시작 진입 | 이전 세션 좀비 (sub-agent 추적 외, 본 세션 시작 이전 발생) | \`.claude/hooks/session-start-zombie-check.sh\` (SessionStart hook) |

**ETIME 30분 임계값 SSoT** — 가드 B / 가드 C / qa.md 좀비 카나리아 3 위치 정합 박제. qa/dev 사이클 1회 이상 경과 → 본 세션 spawn 가능성 매우 낮음.

## verify-zombie-check 회귀 가드 자동화 (메타 가드)

\`scripts/verify-zombie-check.mjs\` (CI \`detect-and-test\` step) 가 SSoT 박제 5건 정합 검증:
1. CLAUDE.md \`#### 가드 A\` sub-section
2. CLAUDE.md \`#### 가드 B\` sub-section
3. qa.md "이전 세션 좀비 카나리아" 항목
4. \`.claude/hooks/session-start-zombie-check.sh\` 파일 존재 + executable bit
5. forensic 보고서 (\`docs/reports/20260510-419-...md\`) 존재

박제 중 하나라도 회귀 시 exit 1 + PR 머지 차단. **가드 자체의 회귀를 자동 감지하는 메타 가드** — 1년 후 다른 변경에서 sub-section 헤더가 의도치 않게 제거돼도 CI 가 즉시 차단.

## 재사용 가능 패턴

본 패턴은 dev 서버 좀비 외에도 동일 적용:
- **cargo test 좀비** (volt #52 의 일반화) — 본 세션 sub-agent 가 spawn 안 했지만 이전 세션 잔존
- **docker container 좀비** (포트/리소스 점유)
- **ssh tunnel 좀비**
- **기타 "sub-agent 가 직접 spawn 안 했지만 환경에 영향 주는 장기 프로세스"**

## 트리거 조건 (본 패턴 적용 판단)

- 본 세션 sub-agent 가 spawn 한 PID 만 추적하는 SSoT 가 있을 때
- 이전 세션 / 사용자 직접 / 외부 도구 spawn 좀비가 환경 공유 (포트 / 파일 락 / IPC) 가능할 때
- 좀비가 단순 점유가 아닌 응답으로 메인을 오인시킬 수 있을 때 (HTTP 응답 / port LISTEN / lock 파일 존재)

## 선행 / 직교 volt

| volt | 관계 |
|------|------|
| #46 (sub-agent background cleanup 누락) | spawned_bg_pids 도입 근거 — 본 #440 의 추적 단위 정의 출처 |
| #52 (cargo 좀비 누적) | bg_process_handoff 도입 근거 — 본 #440 의 트리거 정의 출처 |
| #79 (agent-browser-chrome 좀비 변형) | 도구 wrapper 가 spawn 한 child 좀비 — 본 #440 의 외부 spawn 변형 |
| #24 (sub-agent 마무리 단계 외부 가시성 박제 누락) | sub-agent 보고 신뢰 vs 메인 독립 검증 — 본 #440 의 Gap 2 직접 원인 |

본 #440 은 위 4건의 **"추적 단위 외부 변형"**. SSoT 가 sub-agent 시작 PID 만 커버하는 한계 박제 + 직교 가드 패턴 제시 + 메타 가드 (CI 자동) 확장.

## 출처

- astro-simulator [PR #441](https://github.com/coseo12/astro-simulator/pull/441) (Phase 1 — forensic + 가드 A)
- astro-simulator [PR #442](https://github.com/coseo12/astro-simulator/pull/442) (Phase 2a — 가드 B + qa.md + reviewer 권고 1/2/3)
- astro-simulator [PR #443](https://github.com/coseo12/astro-simulator/pull/443) (Phase 2b — 가드 C + 회귀 가드 + reviewer 권고 흡수)
- astro-simulator [이슈 #440](https://github.com/coseo12/astro-simulator/issues/440)
- forensic 보고서: \`docs/reports/20260510-419-dev-server-zombie-recurrence.md\`
- 발화점 PR: [#437](https://github.com/coseo12/astro-simulator/pull/437) (#419 mount race fix, D-T2 발화)

## 관련 노트/링크

- [volt #46](https://github.com/coseo12/volt/issues/46) (sub-agent background cleanup)
- [volt #52](https://github.com/coseo12/volt/issues/52) (cargo 좀비)
- [volt #79](https://github.com/coseo12/volt/issues/79) (agent-browser-chrome 좀비)
- [volt #24](https://github.com/coseo12/volt/issues/24) (sub-agent 마무리 외부 가시성)
- harness real-lessons "중복 브랜치 dev 서버 오진 방지" 가드 (CLAUDE.md line 284)
