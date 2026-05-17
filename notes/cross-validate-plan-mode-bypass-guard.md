---
title: cross-validate plan-mode 우회 가드 — snapshot+hash 비교 + 자동 롤백 + outcome 스키마 확장
type: knowledge
source_repo: coseo12/astro-simulator
source_issue: 99
captured_at: 2026-05-17
status: refined
tags: [cross-validate, gemini, plan-mode-bypass, snapshot-rollback, outcome-schema, defense-in-depth]
related:
  - ./external-tool-claim-empirical-verify.md
  - ./cross-validate-principle-declaration-bias.md
  - ./cross-validate-dod-self-contradiction.md
  - ./sub-agent-runtime-ssot-variance.md
  - ./self-pin-guard-self-consistency-meta.md
  - ./strict-assertion-vs-fallback-self-contradiction.md
  - ./architect-develop-pull-omission.md
  - ./d2-scope-recontract-pm-architect-orthogonal.md
  - ./meta-risk-bypass-2stage-zero-fire.md
---

## 요약

Gemini CLI 의 cross-validate 호출이 plan-mode 우회 (Gemini 가 의도하지 않은 파일 수정) 사고를 일으킬 수 있다. snapshot (`git ls-files --modified` porcelain + 파일 hash 혼합 D 비교) + 자동 롤백 (R4) + outcome 스키마 3 필드 확장 (`plan_bypass` / `bypass_files` / `rollback_failed`) 의 3축 가드로 차단.

## 본문

**문제**:
- Gemini CLI 의 `--allow-modifications` 명시 없이도 plan-mode 우회 (예상 외 파일 수정) 가 발생할 수 있음
- 정적 후처리 검사로는 "plan-mode 우회" 자체를 검출하기 어려움 — 의도 vs 실제 변경의 차이를 보존해야 함

**가드 설계 (3축)**:

1. **snapshot 비교 D — porcelain + hash 혼합** (`scripts/cross_validate.sh`)
   - cross-validate 호출 직전: `git status --porcelain=v1` + 변경 후보 파일 hash 캡처
   - 호출 직후: 동일 명령 재실행, diff 발생 시 plan-mode 우회 의심
   - porcelain 만으로는 "수정 후 원복" 케이스 놓침 → hash 병행 필수

2. **자동 롤백 R4** (cross_validate.sh)
   - plan-mode 우회 검출 시 `git checkout HEAD -- <bypass_files>` 자동 실행
   - 실패 시 (`rollback_failed: true`) 사용자 수동 개입 필수 — 즉시 사고 보고

3. **outcome 스키마 3 필드 확장** (5 페르소나 .md SSoT)
   - `plan_bypass: bool` — 우회 감지 여부 (메인이 read)
   - `bypass_files: string[]` — 변경된 파일 목록
   - `rollback_failed: bool` — 자동 롤백 성공 여부
   - 5 페르소나 `.md` SSoT 동시 박제 (`architect/developer/pm/qa/reviewer`)

**검증 의무 (메인 오케스트레이터)**:
```bash
scripts/parse-cross-validate-outcome.sh <outcome.json>
# plan_bypass == true 발견 시 즉시 사용자 보고 + bypass_files 추가 검증
```

**격리 동적 테스트 4 케이스** (보안 가드 SSoT):
1. 정상 (clean) → snapshot diff 없음
2. tracked 파일 수정 → rollback 발화 + plan_bypass=true
3. untracked 파일 생성 → 자동 delete + plan_bypass=true
4. set -u 빈 배열 처리 (bash 호환) → 안전

**4중 확증 (메인 + dev + reviewer + qa)**:
- dev D4 회귀 가드 (`plan_bypass=false` 자기 검증)
- reviewer 정적 분석 (스크립트 보안 검증)
- qa 동적 매트릭스 (격리 4 케이스)
- 메인 (outcome.json parse 의무)
- 4 축 결정적 일치 → 자기 박제 가드 self-consistency 입증

**일반화**:
- 외부 도구 (Gemini, Claude API, MCP 서버) 호출 직후 작업 디렉토리 상태 보존이 필수인 경우 본 3축 패턴 (snapshot+hash / 자동 롤백 / outcome 스키마) 재사용
- "도구 호출 결과 + 부수 효과 분리" 의 일반화 — 부수 효과를 보존 + 자동 정리 + 보고로 3축 박제

## 관련 노트/링크

- 사례: astro-simulator [#479](https://github.com/coseo12/astro-simulator/issues/479) (PR #482, 2026-05-15)
- SSoT: 5 페르소나 `.md` outcome 스키마 (volt #58 sub-agent SSoT variance 의 보안 확장)
- 관련 volt: #51 (cross-validate 오탐), #55 (원칙 선언 직후 cross-validate), #66 (cross-validate self-contradiction)
