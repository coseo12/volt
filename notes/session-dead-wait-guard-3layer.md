---
title: 세션 중단 dead-wait 방지 3계층 가드 (heartbeat + SessionStart 훅 + 대기 상태 파일)
type: knowledge
source_repo: coseo12/astro-simulator (구현 원본) → coseo12/harness-setting (기여 대상)
source_issue: 121
captured_at: 2026-07-10
status: refined
tags: [session-lifecycle, dead-wait, ScheduleWakeup, SessionStart-hook, orchestrator, harness-upstream, z-pattern]
related:
  - ../meta/patterns/sub-agent-bg-process-leak-pattern.md
  - ./ssot-tracking-zombie-external-orthogonal-guards.md
  - ../meta/patterns/regression-guard-3stage-simulation.md
  - ./adr-z-pattern-option-a-first-adoption.md
---

## 요약

AI 에이전트 세션(Conductor 등)이 중단·재시작될 때, 메인 오케스트레이터 컨텍스트에는 "대기 중"만 남고 실제 waiter(background watch / sub-agent notification 경로)는 SIGKILL로 소멸한다. 그 결과 **아무것도 모델을 재호출하지 않아 무기한 침묵(dead-wait, 무인지 대기)**하고 사용자는 진행 중으로 오인한다. 작업 유실보다 이 "무인지 대기"가 더 치명적이다. 이를 3계층 직교 방어로 막는 재사용 가능한 가드 패턴. astro-simulator #817(PR #819)에서 구현·검증 완료, harness upstream 기여(Z-패턴 Phase 2) 대상.

## 본문

**근본 원인 (실측 확정)**
- background watch·sub-agent 는 세션의 자식 프로세스 → 세션 재시작 시 SIGKILL 동반 사망(exit 137, 실측 5회+).
- 반면 스케줄러 기반 지연 재호출(Claude Code의 `ScheduleWakeup`)은 **세션 재시작에도 지속 발화**(실측) → **dead-man's switch**로 사용 가능.

**3계층 직교 방어 (우선순위 순)**
1. **fallback heartbeat (1차, 하드 보증)** — 모든 background 대기 진입 시 장기 `ScheduleWakeup`(1200~1800s)을 병행 예약. notification이 먼저 오면 no-op(저비용 상태 재확인), 세션이 죽었다 재개되면 이 wakeup이 **유일한 재호출 신호**가 되어 침묵을 깬다. **단발성이므로 대기 해소 시 재예약하지 않음(자연 종료)** — 명시적 취소 API 불필요.
2. **SessionStart 복구 훅 (2차, 결정적 노출)** — 세션 시작 시 미해소 대기 잔존을 stdout 경고로 노출(exit 0, 블로킹 금지). 모델이 대기 재개 대신 즉시 상태 재확인하도록 유도. 좀비 프로세스 검출 훅과 동형 구조.
3. **대기 상태 파일 (3차, 맥락 상세)** — `.context/pending-waits.json` 등에 `{id: "<kind>:<식별자>", kind: sub-agent|ci-run, description, created_at}` 목록화. 2차 훅이 읽어 노출할 데이터 소스. **파일 write는 best-effort(크리티컬 패스 밖)** — 누락돼도 1차 heartbeat가 침묵을 깬다.

**행동 규약 (메인 오케스트레이터)**
- 기록: 대기 진입 = `ScheduleWakeup 예약 + 상태파일 append`를 **하나의 원자 단위**로. 
- 제거: 대기 해소 시 해당 id 항목 제거.
- 복구 프로토콜(훅 경고를 본 뒤): `(1) 대상 상태 조회(라벨/메시지) → (2) 생사·완료 판단 → (3) 항목 제거 또는 작업 재개`. **대기를 그대로 재개하지 말 것 — waiter는 이미 소멸했을 수 있다.**

**설계 결정 (트레이드오프)**
- 별도 훅 파일 채택(기존 좀비-검출 훅 확장 아님): 단일 책임 분리 + harness update clobber 면역(로직은 non-managed 신규 파일) + 회귀 격리(자체 verify).
- heartbeat 우선/파일 보조: "모델이 매번 파일 write를 성실히 하리란 보장이 약하다"는 근본 취약점을, 침묵 방지 크리티컬 패스를 스케줄러에 두어 완화.
- 훅은 **검출만, 자동 정리 안 함**(masking 방지, fail-visible).

**방어적 처리(cross-validate 반영)**
- **Grace Period**: `created_at` 기준 최소 유예(예: 60s) 경과 항목만 경고 → 세션 종료 직후 재시작 시 방금 진입한 대기 오탐 방지(좀비 검출 훅의 ETIME 임계값과 동형).
- **비정상 timestamp 대칭 노출**: 파싱불가 OR 미래 timestamp = 보수적 노출(은닉 금지).
- **방어적 JSON**: 빈/whitespace 파일 = 정상 초기 상태로 조용히 통과, 진짜 invalid만 손상 경고. 어떤 입력에도 **exit 0 불변**(SessionStart 블로킹 절대 금지) + shell injection 미해석.

**harness generic 정제 사항(upstream 기여 시)**
- 프로젝트 고유 참조 제거: 특정 이슈번호(#817), 프로젝트 경로, 특정 incident(가드 A/B/C ↔ 좀비 incident) 링크를 프로젝트-중립 서술로.
- `.context/` 경로·파일명은 관례로 두되 설정 가능하게(env override 권장 — testability).
- 좀비 검출 훅이 있는 harness라면 "가드 A/B/C(프로세스 라이프사이클)의 직교 확장 = 가드 D(대기 라이프사이클)"로 위치.

## 관련 노트/링크

- 구현 PR: https://github.com/coseo12/astro-simulator/pull/819 (커밋 a54e138)
- 설계 ADR: astro-simulator `docs/decisions/20260710-817-dead-wait-guard.md` (Accepted, cross-validate 통합)
- 이슈: https://github.com/coseo12/astro-simulator/issues/817
- 검증: verify 스크립트 `--self-test` 3중 시뮬(positive→negative→recovery) 17 assertion, hook exit 0 불변 13입력 전수, 3 hook 결합성, injection 면역
- Z-패턴(harness-managed divergent) Phase 2 대상 — settings.json에 훅 등록 1줄만 managed drift, 로직은 non-managed 신규 파일
