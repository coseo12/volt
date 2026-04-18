---
title: sub-agent 마무리 보고 누락 반복 패턴 — 검증은 신뢰하되 박제는 재검증
type: pattern
source_repo: coseo12/astro-simulator
source_issue: 24
captured_at: 2026-04-18
status: refined
tags: [sub-agent, workflow, claude-code-harness, dev-persona, qa-persona, finalization, process]
related:
  - ./pm-sub-agent-multi-turn-round-drift.md
  - ../../notes/github-auto-close-keyword-syntax.md
---

## 배경/상황

astro-simulator P6 마일스톤(2026-04-17) 진행 중, Claude Code harness의 dev/QA 페르소나 sub-agent가 **마무리 단계를 반복적으로 누락**했다. 총 4회 관찰:

1. **P6-B QA (PR #195)** — 동적 검증 보고는 메인 컨텍스트에 했으나 `gh pr comment`로 PR 본문에 박제를 누락. PR 히스토리에만 reviewer 코멘트만 남음.
2. **P6-C QA (PR #197)** — 동일. 매트릭스 결과만 메인 보고, PR 코멘트 박제 누락.
3. **P6-C dev (PR #197)** — ADR + 구현 + 단위 테스트 모두 워킹 트리에 작성했으나 커밋/푸시/PR 생성 전에 세션 종료. 메인이 실제 상태를 확인해 직접 마무리.
4. **P6-D dev (PR #198)** — 보고에 PR URL/커밋 SHA/핵심 수치 누락. "대기 중 다른 작업은 이미 완료 상태"라는 모호한 메시지만 남기고 종료.
5. **P6-E dev (PR #200)** — 4번째 재발. 파일 생성/수정은 모두 완료했으나 커밋/푸시/PR 미수행.

공통점:

- **빌드/테스트/브라우저 검증은 수행**하나 GitHub 상태 변경(커밋/PR/코멘트)에서 이탈
- sub-agent 관점에서는 "본질 작업 완료"로 간주, harness 관점에서는 "외부 가시성 0"
- 메인 컨텍스트가 매번 직접 상태 확인 후 마무리 수행

## 내용

**관찰**:

- dev agent 프롬프트에 이미 "작업 순서 N단계: 커밋 + 푸시 + PR 생성" + "보고에 PR URL/커밋 SHA 포함 필수"를 명시해도 누락 발생
- QA agent는 "`gh pr comment <번호>` 실행 필수" 명시에도 메인 보고만 하고 게시 누락
- 반복 발생하므로 프롬프트 강조만으로는 불충분 — 설계 개선 필요

**추정 원인**:

1. sub-agent 토큰 한계 근접 시 마무리 단계 압축
2. "검증 성공 = 작업 성공"으로 내부 완료 기준 혼동 (외부 가시성 누락)
3. tool 사용 순서 상 중간 단계(빌드/테스트)가 길어지면 마지막 단계의 주의력 저하

**대응 옵션**:

- **A**: 메인 컨텍스트가 sub-agent 보고 수신 후 GitHub 상태(PR/코멘트/라벨) 직접 검증을 루틴화. 누락 시 자동 보완 박제. (현재 방식 — 안전하나 중복 확인 비용)
- **B**: sub-agent 프롬프트 말미에 **"마무리 체크리스트 JSON 반환 필수"** 강제. 각 항목(커밋 SHA / PR URL / 코멘트 URL)을 JSON field로 요구해 누락을 구조적으로 방지.
- **C**: harness 수준에서 sub-agent 종료 후 자동 `git log --oneline -1` / `gh pr list` 비교로 미커밋 파일/미생성 PR 감지.

## 교훈 / 다음에 적용할 점

1. **sub-agent 위임은 "검증"까지는 신뢰하되 "박제"는 신뢰하지 말 것** — 메인 컨텍스트가 GitHub 상태를 직접 확인하는 루틴을 기본으로.
2. **프롬프트에 "PR URL + 커밋 SHA + 테스트 카운트 모두 포함" 명시만으로는 불충분** — 구조화된 반환 포맷(예: JSON) 요구 또는 harness 수준 자동 검증이 필요.
3. **dev 페르소나 sub-agent 최종 단계는 분리 호출 권장** — "구현 + 검증"과 "커밋 + 푸시 + PR"을 별도 sub-agent로 분리하면 각각 책임이 명확해져 누락 감소 기대.
4. **Claude Code `capture-merge`/`create-pr` 같은 전용 스킬이 있는 이유** — 본 관찰이 그 설계의 타당성을 방증. 범용 dev agent보다 작업 유형별 전용 스킬이 마무리 단계 누락을 구조적으로 방지.
5. 메인 오케스트레이터 관점에서는 **sub-agent 보고 → GitHub 상태 확인 → 필요 시 직접 보완** 흐름을 기본값으로 설계.
