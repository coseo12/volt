---
title: YAML 1.1 'on' → True boolean 함정 + 우회 — workflow_dispatch 영구 무효화 변형
type: knowledge
source_repo: coseo12/astro-simulator
source_issue: 102
captured_at: 2026-05-17
status: inbox
tags: [yaml, github-actions, boolean-coercion, on-keyword, workflow-trigger, parser-quirk]
related:
  - ../notes/workflow-dispatch-default-branch-trap.md
  - ../notes/workflow-dispatch-release-acceleration-trap.md
---

## 요약

YAML 1.1 spec 에서 `on` / `off` / `yes` / `no` 가 boolean 으로 자동 coercion. GitHub Actions workflow 의 `on:` 트리거 키가 일부 parser (Python `yaml.safe_load` 폴백 등) 에서 `True` boolean 키로 해석되어 트리거 인식 실패. **CI statusCheckRollup 은 PASS 로 나타나지만 trigger 자체 미발화** — silent 함정 변형 (volt #62 의 GitHub Actions parse-fail 변형).

## 본문

**증상**:
```yaml
# .github/workflows/adr-z-pattern-health.yml
on:        # ← YAML 1.1 spec 에서 True 로 coercion
  schedule:
    - cron: '0 0 1 * *'
  workflow_dispatch:
```

`yaml.safe_load` 가 `{True: {schedule: [...], workflow_dispatch: None}}` 로 파싱 → GitHub Actions 가 트리거 key 미인식 → workflow 영구 무효화. **CI statusCheckRollup PASS** (PR CI 와 무관) 라 PR 머지 후에야 발견.

**우회 (PR #491 #483 hotfix)**:
```yaml
"on":      # ← string 강제 (quote)
  schedule:
    - cron: '0 0 1 * *'
  workflow_dispatch:
```

또는 single quote 도 동일 효과:
```yaml
'on':
  schedule: ...
```

**관련 함정 — 동일 파일명 stale ID** (PR #492 hotfix-2):
- workflow 파일이 한 번 invalid 상태로 인덱싱되면 GitHub 가 ID 를 frozen 처리
- 동일 파일명으로 fix push 해도 stale ID 유지 (404 지속)
- **우회**: 파일명 변경 (`adr-z-pattern-health.yml` → `adr-z-pattern-health-v2.yml`) → 새 ID 발급

**예방**:
- 신규 workflow 파일 PR 도입 시 PR DoD 에 "default branch 머지 후 `gh workflow run <file>` 발화 실측" 명시 (volt #45)
- `yamllint` 또는 `actionlint` pre-commit hook 도입 — `on` 키 boolean coercion 경고
- YAML 1.2 spec 은 boolean coercion 제거하나 GitHub Actions 는 1.1 parser 사용 (2026-05 기준)

**도구별 처리**:
- Node.js `js-yaml` (1.2 기본): `on` 을 string 으로 유지 ✓
- Python `yaml.safe_load` (1.1): `on` 을 `True` 로 coercion ✗
- Go `gopkg.in/yaml.v3` (1.2): `on` 을 string 으로 유지 ✓
- GitHub Actions runner internal: 1.1 spec (2026-05 기준 — 확인 권장)

**일반화**:
- YAML 1.1 boolean coercion 영향: `on`, `off`, `yes`, `no`, `true`, `false`, `True`, `False`, `TRUE`, `FALSE`, `y`, `Y`, `n`, `N`
- 키로 쓸 때 quote 강제 — 값으로 쓸 때도 의도가 string 이면 quote

## 관련 노트/링크

- 사례: astro-simulator PR #491 (hotfix YAML on quote, 2026-05-16) + PR #492 (file rename hotfix-2)
- YAML 1.1 spec: https://yaml.org/type/bool.html
- 관련 volt: #45 (workflow_dispatch 2단계 함정), #97 (workflow_dispatch release 가속 트리거)
