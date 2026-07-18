---
title: AI 환경 전수 감사 → 7이슈 연속 완주 — Z 패턴 3종 세트 사이클 + reviewer 가 메인 결함 2회 적발
type: report
source_repo: coseo12/astro-simulator
source_issue: 126
captured_at: 2026-07-19
status: refined
tags: [ai-env-audit, multi-agent-fanout, z-pattern, upstream-contribution, pipeline]
related:
  - ./adr-z-pattern-option-a-first-adoption.md
  - ./reviewer-cross-validate-5-step-loop.md
  - ./reviewer-qa-gate-skip-headless-self-pass.md
  - ./stack-pr-rebase-force-push.md
  - ./harness-update-rerun-divergent-overwrite.md
  - ./label-taxonomy-skills-layer-propagation.md
  - ./skill-cli-snippet-empirical-check.md
---

## 배경/상황

프로젝트 전수 감사(코드 12이슈 #840~#851 박제)에 이어 AI 에이전트 환경(.claude agents/skills/commands/hooks/settings + .harness)을 별도 전수 감사하고, 발견 7이슈(#852~#858)를 한 세션에서 연속 완주했다 (2026-07-18~19). 다운스트림 PR 6건 머지 + upstream harness-setting PR 5건(#320~#324) + 통합 결함 보고 이슈(#319) 제출.

## 내용

**감사 방법**: 영역별 병렬 general-purpose 에이전트 3개(agents/commands, skills, settings/hooks/manifest) + 메인이 가드 5종 라이브 실행·핵심 주장 직접 실측(3중 확인). 결과 구조가 유의미 — **가드 계층 자체는 전 항목 건강**(SSoT 45체크/데코레이터/훅 정합)했고, 결함은 전부 "외곽"(권한 방어선 0, 스킬 계층 라벨 미전파, 실기 drift, 유령 참조)에 있었다. 가드가 검사하는 범위는 지켜지고, 검사 범위 밖에서 부패가 진행된다는 전형적 패턴.

**완주 사이클**: 이슈별 feature 브랜치 → Z 패턴 Phase 1(데코레이터) → upstream PR 동시 제출(Phase 2, 제목에 다운스트림 이슈 ref — Amendment 10 정합) → reviewer sub-agent (차단 2회 적발: sidecar 사실 오류, eval dead reference — 둘 다 메인이 작성한 결함) → CI → squash 머지 → 수동 close. reviewer 사이클이 메인 산출물의 사실 오류를 2회 잡아낸 것이 게이트 가치의 직접 증거.

**사고 1건과 복구**: upstream 클론이 이전 이슈 브랜치에 남은 상태에서 다음 이슈 커밋을 push → PR #322 에 무관 커밋 오염. 브랜치 분리 + `--force-with-lease=<branch>:<sha>` (명시 lease — shallow clone 은 tracking ref 가 stale 해 기본 lease 실패) 로 복원, PR 커밋 목록 재확인으로 검증. 교훈: **멀티 저장소 왕복 작업 시 커밋 직전 `git branch --show-current` 확인 의무**.

**기타**: CI detect-and-test 가 PR 당 ~17분 (dev 서버 10회 cold-boot — astro-simulator#846 개선 대기) × 6 PR — 폴링 주기 270s 규약이 잘 맞았다. branch protection 은 PR 필수 + enforce_admins 만 1단계 적용 (required checks 는 paths-filter 영구 대기 위험으로 유예 박제).

## 교훈 / 다음에 적용할 점

1. **감사→이슈→완주 파이프라인의 단위**: 이슈당 "다운스트림 PR + upstream PR + 데코레이터 cross-link" 3종 세트를 한 사이클로 묶으면 Z 패턴이 자연 완결된다 — Phase 2 를 미루면 TODO aging 만 쌓인다
2. **reviewer 는 메인 산출물에도 필수** — 차단 2건 모두 메인의 "그럴듯한" 서술/스니펫이었다. 자기 산출물 사실 오류는 자기 검토로 안 잡힌다
3. 멀티 저장소 세션에서 클론의 브랜치 상태는 매 커밋 전 확인 (사고 1 재발 방지)
4. 가드 건강 ≠ 환경 건강 — 감사는 "가드가 검사하지 않는 외곽" 을 우선 훑어야 한다

## 관련 노트/링크

- https://github.com/coseo12/harness-setting/issues/319 (통합 결함 보고)
- upstream PR: harness-setting#320~#324 / 다운스트림: astro-simulator#852~#858, PR #859~#864
