---
title: harness update --apply-all-safe 재실행 함정 — divergent 로컬 해시 기록이 2차 실행의 조용한 덮어쓰기 유발
type: knowledge
source_repo: coseo12/astro-simulator (PR #859, 이슈 #853) / coseo12/harness-setting#319
source_issue: 123
captured_at: 2026-07-19
status: refined
tags: [harness-update, manifest, divergent, silent-overwrite, z-pattern]
related:
  - ./manifest-state-recovery-pattern.md
  - ./harness-frozen-file-split-pattern.md
  - ./adr-z-pattern-option-a-first-adoption.md
  - ./session-dead-wait-guard-3layer.md
  - ./ai-env-audit-7issue-z-pattern-cycle.md
---

## 요약

`harness update --apply-all-safe` 를 divergent 파일이 잔존한 상태에서 같은 세션에 2회 실행하면, 1차 실행이 미해결 divergent 파일들의 **로컬 해시를 매니페스트에 기록**하기 때문에 2차 실행이 이를 "사용자 미수정(pristine)"으로 오분류해 **upstream 판으로 조용히 덮어쓴다**. 다운스트림 전용 가드(settings.json hook 등록, qa.md 다운스트림 불릿, harness-guards.yml concurrency 등)가 소리 없이 소실된다.

## 본문

**실측 시퀀스 (astro-simulator v4.2.5→v4.4.0 동기화, 2026-07-18)**:
1. 1차 `--apply-all-safe`: 신규 1 + 안전 4 적용, divergent 6건은 "미해결 — --interactive 필요" 안내로 보존. 단, 매니페스트에는 divergent 파일의 **로컬 해시**가 기록됨
2. 이후 divergent 중 일부(dead-wait 파일쌍)를 upstream 판으로 수동 채택 → 매니페스트 로컬 해시와 불일치 → 신규 drift 로 오탐
3. 재조정 목적의 2차 `--apply-all-safe`: 나머지 divergent 4파일(settings.json/qa.md/developer.md/harness-guards.yml)이 "pristine"으로 오분류되어 **upstream 판으로 덮어써짐** — settings.json 의 zombie-check hook 등록 제거, qa.md 다운스트림 가드 3종 삭제, harness-guards.yml concurrency/create-pr SSoT 가드/lib skip 조건 삭제
4. `git checkout HEAD -- <4파일>` 복원 + 데코레이터/sidecar 재정비로 해소

**재발 방지**: divergent 잔존 상태에서 apply 재실행 금지 — 재실행 전 `--check` 로 divergent 목록이 비었는지 확인. upstream 개선 후보: divergent 파일은 매니페스트에 divergent 마킹을 보존하거나 로컬 해시 기록을 하지 않는 방식.

volt #27(부분 실패 교착)과 동류의 "매니페스트-파일 비원자성" 클래스지만, 이번 건은 **정상 실행 2회의 조합**만으로 발생하는 변형이라는 점이 다르다.

## 관련 노트/링크

- https://github.com/coseo12/harness-setting/issues/319 (§3 보고)
- https://github.com/coseo12/astro-simulator/issues/853 (실측 박제 코멘트)
- volt #27, #28 (인접 클래스)
