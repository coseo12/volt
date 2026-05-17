---
title: YAML block scalar + bash heredoc indent 함정 — body 0 column indent 시 workflow 등록 실패
type: knowledge
source_repo: coseo12/astro-simulator
source_issue: 103
captured_at: 2026-05-17
status: refined
tags: [yaml, block-scalar, bash-heredoc, indentation, github-actions, parser-quirk]
related:
  - ./downstream-structure-compat-checklist.md
  - ./yaml-on-boolean-coercion-trap.md
---

## 요약

GitHub Actions workflow YAML 의 `run:` step 안에 bash heredoc 본문 (`<<HEREEND ... HEREEND`) 을 박을 때, heredoc body 가 0 column 에서 시작하면 YAML block scalar parser 가 step 종료로 오인 → workflow 등록 실패. **rendered body 의도는 유지하되** 10 space indent prepend 로 우회.

## 본문

**증상** (qa 단계 차단 사례):

```
- name: Create issue
  run: |
    gh issue create \
      --body "$(cat <<HEREEND
## 배경
자동 탐지 workflow 발화. 임계값 충족.
HEREEND
)"
```

YAML parser:
- `run: |` block scalar 인덴트 기준 = 처음 비공백 라인의 column
- heredoc body 의 `## 배경` 이 column 0 에서 시작 → block scalar 종료 + step 종료 + 후속 YAML 키 (`## 배경` 을 next step 으로 해석 시도)
- 결과: workflow 파일 invalid → GitHub Actions 등록 실패 (silent 함정 — CI 가 워크플로 파일 검증을 별도로 하지 않으면 PR PASS 유지)

**우회 (10 space indent prepend)**:

```
- name: Create issue
  run: |
    gh issue create \
      --body "$(cat <<HEREEND
          ## 배경
          자동 탐지 workflow 발화. 임계값 충족.
          HEREEND
    )"
```

bash heredoc 의 `<<HEREEND` 는 indent 보존 (`<<-HEREEND` 는 leading tab 만 제거). 본문에 10 space 가 그대로 박혀 issue body 로 출력되지만 마크다운 렌더링은 leading whitespace 무시 → **rendered 의도 유지** ✓.

**대안 (검토 후 미채택)**:
1. `<<-` + tab indent — tab 만 제거하므로 mixed indent 위험
2. external 파일 (`issue-body.md`) 참조 — workflow 자체 검증 부담 + 동적 변수 보간 어려움
3. one-liner string concat — 가독성 매우 떨어짐

**예방**:
- workflow YAML 에 heredoc + 마크다운 body 박제 시 indent 의무 (10 space 또는 부모 step indent 보존)
- `actionlint` 또는 `yamllint` strict mode CI 가드 — block scalar indent 검증

## 관련 노트/링크

- 사례: astro-simulator PR #490 (#483 ADR Z 패턴 자동 탐지, qa 차단 → fix `5272f61`, 2026-05-16)
- 관련 volt: #62 (다운스트림 harness update 부합성 체크리스트 — GitHub Actions parse-fail 변형), #102 (YAML 1.1 on→True boolean)
