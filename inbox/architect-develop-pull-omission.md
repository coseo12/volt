---
title: architect develop pull 누락 사고 — 옛 스크립트 사용으로 가드 첫 실전 검증 누락
type: knowledge
source_repo: coseo12/astro-simulator
source_issue: 108
captured_at: 2026-05-17
status: inbox
tags: [architect, sub-agent, develop-pull, worktree-drift, stale-script, dev-handoff]
related:
  - ../notes/architect-worktree-base-drift-sync.md
  - ../notes/cross-validate-plan-mode-bypass-guard.md
  - ../notes/self-pin-guard-self-consistency-meta.md
---

## 요약

architect sub-agent 가 develop 동기화 안 한 상태로 호출되어 **옛 cross_validate.sh 사용** → #479 가드 (snapshot+hash 비교) 첫 실전 검증 누락. dev 단계가 develop pull 후 새 스크립트로 보완. **architect 호출 직전 develop pull 의무** — sub-agent worktree 격리의 부수 함정.

## 본문

**사고 시퀀스 (PR #486 #476 Amendment 1, 2026-05-16)**:

1. PR #482 (#479 cross-validate plan-mode 우회 가드) 머지 직후 develop 에 `scripts/cross_validate.sh` 신규 박제 (snapshot+hash 비교 + 자동 롤백 + outcome 스키마 확장)
2. PR #486 architect sub-agent 호출 (Gemini cross-validate 적용 의무)
3. architect sub-agent worktree 가 develop fetch 안 한 상태 (기존 worktree drift) → 옛 `cross_validate.sh` 호출
4. outcome.json 에 `plan_bypass` / `bypass_files` / `rollback_failed` 필드 부재 → 가드 첫 실전 검증 **건너뛰어짐**
5. dev sub-agent 가 develop pull 후 새 스크립트로 보완 → outcome 4 필드 결정적 일치 (메인/dev/reviewer/qa) 확인

**근본 원인**:
- architect sub-agent 가 `git fetch origin develop && git pull --ff-only origin develop` 절차 누락
- worktree 격리 (volt #84) 가 develop tip 동기화 의무를 자동화하지 않음
- "최신 develop = 최신 가드" 의 의무 박제 부재

**예방 (architect.md SSoT 박제 후보)**:
```markdown
## 절차

1. **develop 동기화 의무** (sub-agent 호출 직후 최우선):
   ```bash
   git fetch origin develop
   git pull --ff-only origin develop  # 충돌 시 main 오케스트레이터에 보고 + sub-agent 종료
   ```
2. ...
```

**보강** — sub-agent 마무리 체크리스트 JSON 에 develop pull 확인 필드 추가:
```json
{
  "develop_pulled": true,
  "develop_tip_sha": "5209c21",
  ...
}
```

**일반화**:
- sub-agent 호출 시 worktree drift 가 신규 가드 실전 검증 누락의 근본 원인
- 신규 가드 도입 PR 머지 후 1~2 PR 의 architect sub-agent 호출이 가장 위험 (옛 스크립트 사용 빈도 높음)
- harness real-lessons "sub-agent worktree drift" (volt #84) 의 가드 측 변형

**복구 가능성**:
- dev 단계가 develop pull 후 보완 가능 (본 사례) — 다행
- 그러나 만약 reviewer / qa 도 옛 스크립트 사용 시 4중 확증 자체가 무력화

## 관련 노트/링크

- 사례: PR #486 (#476 Amendment 1, 2026-05-16) → dev 보완으로 4중 확증 회복
- 관련 volt: #84 (sub-agent worktree drift), #99 (#479 cross-validate plan-mode 우회), #100 (self-consistency 메타)
