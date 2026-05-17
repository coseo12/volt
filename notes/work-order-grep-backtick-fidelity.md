---
title: 작업 의뢰서 grep 백틱 정합성 — 5 페르소나 SSoT 박제 시 의뢰서-실파일 정확 일치 의무
type: knowledge
source_repo: coseo12/astro-simulator
source_issue: 111
captured_at: 2026-05-17
status: refined
tags: [work-order, sub-agent-handoff, ssot-fidelity, grep-backtick-consistency, dev-precision]
related:
  - ../meta/patterns/regression-guard-3stage-simulation.md
  - ./bash-script-isolation-dynamic-test.md
  - ./sub-agent-runtime-ssot-variance.md
---

## 요약

PM/architect 가 작성한 작업 의뢰서 (PR 본문 / sub-agent 호출 프롬프트) 에 박제할 SSoT 문구를 인용할 때 **백틱 정확 일치 의무**. 의뢰서의 backtick 인용과 실 파일 박제 문구가 한 글자라도 다르면 dev sub-agent 가 잘못된 문구로 박제 → 5/5 SSoT drift 자동 발화.

## 본문

**문제 사례 (5 페르소나 SSoT 박제 PR #481, 2026-05-15)**:

PM 의뢰서 (PR 본문):
```
- **PR 생성 시 반드시 \`create-pr\` 스킬 사용** — ...
```

5 페르소나 .md 박제 (dev 가 PM 의뢰서 그대로 복사):
```
- **PR 생성 시 반드시 `create-pr` 스킬 사용** — ...
```

verify-create-pr-ssot.sh (`grep -nF`) 매칭:
- 의뢰서: `\`create-pr\`` (백슬래시 백틱 — PR 본문 markdown 이스케이프)
- 실 파일: `` `create-pr` `` (raw 백틱)
- → `grep -nF` 가 escaped 백틱 매칭 시 raw 백틱 못 잡거나 그 반대 → drift 발화

**해결**:
- PM 의뢰서에 박제할 SSoT 를 raw text 로 명시 (이스케이프 문법 미혼용)
- 또는 의뢰서에 박제 후 dev 가 raw text 추출 의무 (이스케이프 제거)
- 또는 verify-*.sh 스크립트의 매칭 패턴이 이스케이프 변형까지 흡수 (단, 가드 약화 위험)

**의의**:
- PR 본문 markdown 렌더링 vs raw file 박제 의 차이를 의식적으로 박제
- "작업 의뢰서 = 박제 대상의 SSoT" 의무 — 의뢰서 잘못되면 박제 결과 자동 drift
- 5 페르소나 SSoT 박제 워크플로의 미세 함정

**일반화 — sub-agent handoff 정확성**:
- PM/architect → dev 호출 시 박제 문구는 **raw text 박스 (` ```text ... ``` `) 인용** 의무
- markdown inline 백틱 인용 (`` ` ``) 금지 — 이스케이프 변형 위험
- 의뢰서 작성 시 "정확 박제 문구" 박스를 별도 박제

**예방 (PM/architect.md SSoT 박제 후보)**:
```markdown
## 절차

3. **작업 의뢰서 작성 시 박제 문구 raw text 박스 의무**:
   ```text
   <박제 대상 문구 — 백틱 / 이스케이프 / 특수 문자 모두 raw 로 인용>
   ```
   - markdown inline 백틱 인용 (`` ` ``) 금지 — 이스케이프 변형 → verify-*.sh 매칭 실패 위험
```

## 관련 노트/링크

- 사례: PR #481 (#477 5 페르소나 SSoT, 2026-05-15)
- 관련 volt: #96 (3중 시뮬레이션), #98 (격리 동적 테스트), #58 (sub-agent SSoT variance)
