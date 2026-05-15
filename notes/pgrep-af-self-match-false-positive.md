---
title: pgrep -af 자기 매치 false-positive — 좀비 카나리아 검증법 보강
type: knowledge
source_repo: coseo12/astro-simulator
source_issue: 94
captured_at: 2026-05-13
status: refined
tags: [pgrep, ps, 좀비 검증, agent-browser, false-positive, volt-79]
related:
  - ../meta/patterns/agent-browser-chrome-zombie-pattern.md
  - ../meta/patterns/sub-agent-finalization-miss-pattern.md
---

## 요약

`pgrep -af "<pattern>"` 은 명령어 인자에 패턴 자체가 포함되어 자기 자신을 매치하는 false-positive 를 일으킨다. 좀비 카나리아 검증 절차에선 `ps auxww | grep "<pattern>" | grep -v grep` 또는 `pgrep -af "[<첫글자>]<나머지>"` 형태가 정확하다.

## 본문

#### 증상

PR #457 qa sub-agent 복귀 직후 메인 카나리아 (volt #79 agent-browser-chrome 좀비 정리 가드) 실행:

```bash
pgrep -af "agent-browser-chrome-"
# 출력: 63238 63243  (또는 63501, 63672 — 매번 다름)
```

`ps -p 63238,63243 -o pid,etime,command` 로 확인 시 PID 이미 사라짐 (`ps -p` 빈 결과).
재실행하면 또 다른 PID 가 잡힘. 마치 무한 좀비 spawn 처럼 보임.

#### 원인

`pgrep -af` 의 `-f` 옵션은 **명령어 인자 전체** 를 매치 대상으로 삼는다. `pgrep -af "agent-browser-chrome-"` 호출 자체의 인자열 (`pgrep` 자신의 argv) 에 `"agent-browser-chrome-"` 패턴이 포함되므로 pgrep 자기 자신이 매치 결과에 포함된다. pgrep 종료 후 PID 가 사라져 `ps -p` 가 빈 결과 반환.

#### 정확한 검증법

**옵션 A** (간단, POSIX 호환):
```bash
ps auxww | grep "agent-browser-chrome-" | grep -v grep
# grep 자기 자신 제외. 결과 0줄 = 좀비 없음
```

**옵션 B** (pgrep 그대로 사용, regex trick):
```bash
pgrep -af "[a]gent-browser-chrome-"
# 정규식 첫 글자에 character class — pgrep 자기 자신 인자열은 [a] 리터럴이라 매치 안 됨
```

**옵션 C** (process group 격리):
```bash
ps -eo pid,etime,command | awk '/agent-browser-chrome-/ && !/awk/'
# awk 자기 매치 제외
```

#### 영향

- 본 사례: 실제 좀비 0건이었으나 검증 절차 자체의 false-positive 로 "잔존" 오인 → 메인이 KILL 송신 반복 → 매번 다른 PID 가 또 나타나 무한 루프 우려
- 검증 명령이 정확해야 진짜 좀비 vs 검증 노이즈를 구분 가능
- volt #79 agent-browser-chrome 좀비 정리 가드의 **검증 절차 자체** 보강 필요 — "정리 명령" + "검증 명령" 분리 + 검증 명령 false-positive 방지

#### 관련 SSoT

- volt #79 — agent-browser-chrome 좀비 정리 가드 본체. 본 발견은 검증 절차 보강
- volt #24 — sub-agent 박제 검증 누락 패턴. 본 발견은 **도구 자체 함정** 차원 (sub-agent 가 누락한 게 아니라 검증 도구가 잘못된 신호 반환). 직교 차원

## 관련 노트/링크

- [volt #79](https://github.com/coseo12/volt/issues/79) — agent-browser-chrome 좀비 정리 가드
- [volt #24](https://github.com/coseo12/volt/issues/24) — sub-agent 마무리 보고 누락 반복 패턴
- 적용 위치 후보: harness `.claude/agents/qa.md` "좀비 카나리아 항목" + 메인 오케스트레이터 가드 B/C 명령 갱신
