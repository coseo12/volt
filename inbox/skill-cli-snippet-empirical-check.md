---
title: 스킬 CLI 스니펫 실기 대조 의무 — 부재 커맨드는 상시 WARN, 부재 식별자는 silent no-op 오탐
type: knowledge
source_repo: coseo12/astro-simulator (이슈 #856, PR #862) / coseo12/harness-setting#322
source_issue: 125
captured_at: 2026-07-19
status: inbox
tags: [skill-snippet, cli-version-drift, silent-no-op, always-warn, agent-browser]
related:
  - ../notes/agent-browser-eval-console-spy-limit.md
  - ../notes/external-tool-claim-empirical-verify.md
  - ../meta/patterns/agent-browser-chrome-zombie-pattern.md
  - ../notes/label-taxonomy-skills-layer-propagation.md
  - ../notes/ai-env-audit-7issue-z-pattern-cycle.md
---

## 요약

스킬 문서의 CLI 스니펫은 도구 버전과 **실기 대조 없이는 두 가지 오탐 클래스를 생산**한다: (1) 부재 서브커맨드를 조건 검사에 쓰면 **상시 WARN 오탐** (`agent-browser doctor` — 존재하지 않아 항상 실패 → 정상 환경에서도 "미설치" 경고), (2) 부재 플래그/식별자는 **silent no-op** (`--viewport`/`--script` 는 실행 불능, `__solarScene?.camera` optional chaining 은 실패를 삼켜 freeze 탐지가 상시 통과). 스킬 스니펫 도입·수정 시 실기 1회 실행이 검증의 최소 단위다.

## 본문

**실측 (agent-browser 0.21.0, 2026-07-18~19)**:
- `agent-browser doctor` → Unknown command exit 1 → check-browser.sh 가 Chrome 정상 설치 환경에서도 상시 `[WARN] Chrome for Testing 미설치` — alert fatigue 로 신호 무의미화
- `--viewport 375 812` / `--script x.mjs` 플래그 부재 → 반응형·픽셀 검증 절차가 문서상으로만 존재. 올바른 형태는 `set viewport <w> <h>` 서브커맨드 / 독립 node 스크립트
- **동일 클래스 재도입을 reviewer 가 적발**: 수정 PR 에서 내가 쓴 eval 예시 `window.__solarScene?.camera` 가 (a) 핸들에 camera 필드 부재 (b) dev 전용 노출 — optional chaining 이 실패를 삼켜 freeze 탐지 before/after 항상 동일 = 상시 오탐. `window.__simCore?.scene?.activeCamera` (prod 노출 SSoT) 로 정정

**교훈 (agent/skill 개선 반영)**:
1. 스킬 스니펫의 커맨드/플래그/전역 식별자는 **도입·수정 시 실기 1회 실행** 이 DoD (help 대조 포함) — "그럴듯한 CLI 형태" 는 버전 간 계속 drift 한다
2. optional chaining + 조건 가드 (`x?.y`, `if (c)`) 는 dead reference 를 **성공처럼 보이게** 만든다 — 검증 스니펫에서는 fail-visible 말미 표현식이 안전
3. 조건 검사에 쓰는 서브커맨드가 죽으면 검사가 "항상 같은 방향" 으로 오탐한다 — 상시 WARN 은 상시 PASS 만큼 위험 (신호 무의미화)

volt #88 (eval console spy false positive), #79 (Chrome 좀비) 와 직교하는 제3 클래스: **스니펫-도구 버전 정합**.

## 관련 노트/링크

- https://github.com/coseo12/astro-simulator/pull/862 (리뷰 사이클에 재도입 적발 기록)
- https://github.com/coseo12/harness-setting/pull/322
- volt #88, #79 (인접 클래스)
