---
title: agent-browser Chrome 좀비 누적 — sub-agent 마무리 누락 패턴의 agent-browser 확장 (volt #52 연장선)
type: pattern
source_repo: coseo12/astro-simulator (R2 사이클 #361 / PR #363 / PR #365)
source_issue: 79
captured_at: 2026-04-28
status: refined
tags: [agent-browser, process-leak, sub-agent-handoff, qa-agent, browser-test, ci-routine]
related:
  - ./sub-agent-bg-process-leak-pattern.md
  - ./sub-agent-finalization-miss-pattern.md
  - ../../notes/stale-dev-server-port-collision.md
---

## 배경/상황

2026-04-28 R2 후속 PR #365 (baseline 갱신 + ADR 정정) 머지 진행 중 사용자가 "무언가 CPU 를 점유한다" 고 보고. `ps auxww | sort -nrk 3,3 | head -15` 실측 결과 상위 15 프로세스가 모두 `agent-browser-chrome-<UUID>` user-data-dir 사용 Chrome Helper (Renderer / gpu-process). CPU 점유 60%+ × 15 = 800%+, 합계 52 좀비 / 6 세션 / 3일치 누적.

본 발견은 volt #52 (cargo / next dev 좀비) 의 agent-browser 확장 — sub-agent 가 `run_in_background` 또는 외부 도구로 띄운 long-running 프로세스를 **세션 종료 시 정리하지 않고 인계 누락** 하는 동일 패턴.

## 내용

#### 실측 데이터 (2026-04-28 12:00 KST)

```
좀비 카운트: 52 (TERM 후 6 main 잔존 → KILL 후 0)
유니크 user-data-dir: 6 (= 6 sub-agent 세션)
CPU 점유: 60%+ × 15 프로세스 = ~800%+ (시스템 전체)

PID 분포 (시작 시각별):
- 3일 전 (금 03오후): PID 51872, 51885, 52157, 52162, 52969, 52977, 53077, 53085, 53597, 53603 ...
- 이틀 전 (토 10오후): PID 9989, 43064, 43065, 43066, 43067, 43068, 43069 ...
- 오늘 새벽 (2:48 KST): PID 27449, 27450, 27451, ...

가장 누적 큰 좀비:
- PID 52977 (금 03오후, gpu-process): CPU 누적 453:42.35
- PID 43069 (토 10오후, renderer): CPU 누적 379:39.98
- PID 9989 (월 08오후, renderer): CPU 누적 377:36.85
```

#### 식별자

각 좀비 세션은 `--user-data-dir=/var/folders/.../T/agent-browser-chrome-<UUID>` 인자로 식별 가능. 사용자 본 Chrome 과 명확히 분리됨 — 안전한 타겟팅 가능:

```bash
# 안전한 타겟팅 (사용자 본 Chrome 영향 0)
pkill -TERM -f "agent-browser-chrome-"
sleep 3
pkill -KILL -f "agent-browser-chrome-"
```

#### 트리거 (본 세션 내 sub-agent)

본 세션의 R2 사이클에서 qa sub-agent (PR #363) 가 보고한 검증 환경:

> "real Chrome (agent-browser channel: chrome) 1280×720 + 375×667. Level 1 정적 / Level 2 인터랙션 / Level 3 흐름 모두 PASS, 콘솔 에러 0. 스크린샷 7장 `~/Desktop/qa-pr363/` 박제"

`agent-browser` 도구 사용 후 **세션 종료 시 Chrome 인스턴스 명시 정리 누락**. sub-agent 의 마무리 체크리스트 JSON `spawned_bg_pids: []` + `bg_process_handoff: "sub-agent-confirmed-done"` 으로 보고했으나 실제로는 6 헬퍼 프로세스 잔존.

#### CLAUDE.md SSoT 의 갭

CLAUDE.md 의 9 필드 SSoT (`spawned_bg_pids` / `bg_process_handoff`) 는 sub-agent 의 `run_in_background=true` 로 띄운 프로세스를 커버하나, **agent-browser 도구가 내부적으로 spawn 하는 Chrome Helper 프로세스 (gpu-process / renderer / 등)는 sub-agent 가 인지하지 못해 PID 추적 불가**. agent-browser 의 정리는 도구 자체의 lifecycle 책임이지만, 비정상 종료 / sub-agent timeout / SIGKILL 시 좀비 잔존 가능성이 구조적으로 존재.

#### 선행 이슈와의 관계

- volt #46 (`spawned_bg_pids` / `bg_process_handoff` SSoT 도입): 본 패턴의 사전 가드 — 단 agent-browser 미커버
- volt #52 (cargo / next dev 좀비, run_in_background 누락): 동일 패턴의 다른 도구 버전 — 본 이슈는 agent-browser 변형
- CLAUDE.md `## sub-agent 검증 완료 ≠ GitHub 박제 완료` §"중복 브랜치 dev 서버 오진 방지" — feature 브랜치 dev 서버 잔존 가드와 동일 카테고리

## 교훈 / 다음에 적용할 점

#### 1. agent-browser 사용 sub-agent 의 마무리 절차 명문화

`.claude/agents/qa.md` / `.claude/agents/browser-test.md` 등 agent-browser 사용 가능 페르소나의 마무리 체크리스트에 다음 단계 추가:

```bash
# sub-agent 반환 직전 의무 — agent-browser 정리
pkill -TERM -f "agent-browser-chrome-"
sleep 2
pgrep -f "agent-browser-chrome-" >/dev/null && pkill -KILL -f "agent-browser-chrome-"
```

#### 2. 메인 오케스트레이터 검증 루틴 확장

CLAUDE.md `## sub-agent 검증 완료 ≠ GitHub 박제 완료` §"중복 브랜치 dev 서버 오진 방지" 처럼 sub-agent 복귀 직후 검증 단계에 추가:

```bash
# agent-browser 좀비 자동 검출
pgrep -af "agent-browser-chrome-" || true
# 발견 시 사용자 보고 + 정리 권한 요청
```

#### 3. SSoT 9 필드 보강 (선택)

`bg_process_handoff` 필드의 enum 값에 `"agent-browser-cleanup-required"` 추가 — agent-browser 사용 sub-agent 가 명시적으로 메인에 정리 책임 인계. 단 agent-browser 가 자체 정리하면 정상 case 가 더 흔하므로 enum 추가 ROI 미달 가능 — `pgrep` 검증으로 충분할 수 있음.

#### 4. agent-browser 도구 자체 개선 권고 (upstream)

agent-browser 도구가 timeout / sub-agent 비정상 종료 시 자체 cleanup hook 추가 (Node `process.on('SIGTERM')` / atexit 등). 본 patten 의 근본 해결.

#### 5. 정기 정리 스크립트 (`harness doctor` 통합 후보)

```bash
# scripts/cleanup-agent-browser-zombies.sh
#!/usr/bin/env bash
ZOMBIES=$(pgrep -f "agent-browser-chrome-" | wc -l | tr -d ' ')
if [ "$ZOMBIES" -gt 0 ]; then
  echo "[warn] $ZOMBIES agent-browser zombies detected"
  pgrep -af "agent-browser-chrome-" | head -5
  exit 1
fi
exit 0
```

`harness doctor` 의 진단 항목에 통합 → 신규 세션 시작 시 자동 검출.

## 일반화 가능 교훈

**도구가 내부적으로 spawn 하는 child process 의 lifecycle 추적 갭** — sub-agent 의 PID SSoT (`spawned_bg_pids`) 가 직접 spawn 한 프로세스만 커버하고, 도구 (agent-browser / playwright / cypress 등) 가 wrapper 로 띄운 child process 는 미커버. 도구 wrapper 의 자체 cleanup 이 정상 case 에서 작동해도 비정상 종료 (SIGKILL / timeout / panic) 시 lineage 가 끊긴 좀비가 잔존.

**메인 오케스트레이터의 검증 루틴에 도구별 child process 패턴 가드 추가** 가 가장 ROI 높음 — `pgrep` 한 줄로 자동 검출 가능.
