---
title: sub-agent 마무리 단계 이탈 + worktree base drift 동시 재현 — 페르소나 공통 패턴
type: pattern
source_repo: coseo12/astro-simulator
source_issue: 82
captured_at: 2026-05-06
status: refined
tags: [sub-agent, worktree, architect, pm, qa, ssot-9-fields, volt-24, isolation]
related:
  - ./sub-agent-finalization-miss-pattern.md
  - ../../notes/architect-worktree-base-drift-sync.md
---

## 배경/상황

#362 architect sub-agent 호출 (R1 ADR Amendment v2 박제) 에서 메모리에 박제된 두 패턴이 **동시 재현**:

1. **sub-agent 멈춤 패턴** (메모리 #411 line 57 → astro-simulator CLAUDE.md 박제) — sub-agent 가 cross-validate / 외부 호출 직전 단계에서 self-report 후 보고 종결. 마무리 체크리스트 JSON (SSoT 9 필드) 미반환. SendMessage deferred tool 환경 미제공 → 메인 직접 인계 필수
2. **worktree base drift 패턴** (메모리 #399 line 36 변형) — Agent tool `isolation: "worktree"` 가 develop tip 이 아닌 옛날 커밋에서 분기됨. 본 사례에서 메인 develop tip = `a77ab41` (2026-05-04) vs worktree base = `12119f3` (2026-04-28, v0.15.0 머지 커밋, 8일 전)

두 패턴이 서로 강화: 멈춘 sub-agent 의 작업물을 살리려 해도 worktree 가 stale 이라 16,000 lines 차이 발생 → 폐기 + 메인 cwd fresh 브랜치 분기 필수.

페르소나 공통: pm (#411 사례), architect (본 #362 사례), qa (메모리 line 53 #421 사례) 모두 관찰. 단발이 아닌 **반복 패턴**.

## 내용

**관찰**:

- sub-agent 가 cross-validate Gemini 응답 대기 중 "Gemini 응답을 기다리는 동안 추가 작업은 보류" 보고로 종료. 마무리 JSON 미반환
- worktree path: `.claude/worktrees/agent-<id>`, branch: `worktree-agent-<id>`. base 가 develop tip 이 아닌 다른 커밋
- worktree 의 R1 ADR 변경분이 어설픈 미완료 상태 (line 307 정정 시도 + line 505 부근 비-범위 섹션 중복 박제)
- 메인 cwd `git status` 는 clean, develop tip `a77ab41` 정상. worktree 폐기 시 영향 0

**조치**:

```bash
# worktree 폐기 (lock 강제 해제 필요)
git worktree remove -f -f .claude/worktrees/agent-<id>
git branch -D worktree-agent-<id>

# 메인 cwd fresh 브랜치
git checkout -b architect/<issue>-<topic>
# 처음부터 재작업 — sub-agent 어설픈 변경분은 참고 가치 낮음
```

**메모리 박제 (CLAUDE.md 학습 적용 시)**:

- "sub-agent 검증 완료 ≠ GitHub 박제 완료" 가드 (volt #24) 의 페르소나 공통 확장 — pm/architect/qa 모두 *외부 호출 직전 단계* 에서 self-report 후 종결
- worktree base drift 가 발생하는 이유는 Agent tool isolation 옵션의 base 결정 메커니즘 미문서화. develop tip 이 아닌 메인 cwd 의 HEAD 또는 fixed 시점 (예: agent 정의 시점) 에서 분기 가능성

## 교훈 / 다음에 적용할 점

1. **sub-agent 호출 프롬프트에 마무리 JSON 의무 명시 강화** — 단순 "마무리 체크리스트 JSON 반환 의무" 외에 "cross-validate / 외부 호출 결과를 박제하지 못한 채 종결 금지" 명시
2. **메인 오케스트레이터 루틴** — sub-agent 보고 수신 시 마무리 JSON SSoT 9 필드 부재 검출 즉시 (a) 재호출 없이 메인 직접 인계 결정 + (b) 작업물 회수 가치 평가 (worktree base drift 검증 의무)
3. **Agent tool isolation 옵션 검증** — sub-agent 호출 직후 worktree HEAD 검증 (`git log -1 --oneline` + `git rev-parse HEAD` 비교 vs 메인 develop tip). drift 시 메인 cwd 진행 결정
4. **agents/*.md 페르소나 SSoT 강화** — pm.md / architect.md / qa.md 모두 §"마무리 체크리스트 JSON 반환 (필수)" 섹션에 "cross-validate / Gemini / 외부 도구 호출은 결과 박제까지 sub-agent 책임 (응답 대기 중 종결 금지)" 명시
5. **worktree drift 발견 시 표준 회수 절차** — sub-agent 어설픈 변경분 폐기 + 메인 cwd fresh 브랜치 + 처음부터 재작업 (sub-agent 작업물 cherry-pick 비용 > 폐기 + 재작업)

본 패턴은 **페르소나 무관 공통** — pm/architect/qa 어떤 sub-agent 라도 cross-validate / 외부 호출 직전 단계에서 발생 가능. 따라서 페르소나별 가드가 아닌 **harness 레벨 가드** (Agent tool 자체 또는 메인 오케스트레이터 후크) 박제 검토 필요.
