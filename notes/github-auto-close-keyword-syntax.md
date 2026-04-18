---
title: GitHub auto-close 파싱 — `Closes: #A, #B` 콜론 문법은 #B 미인식
type: knowledge
source_repo: coseo12/harness-setting (PR #108 v2.14.0 커밋 메시지)
source_issue: 41
captured_at: 2026-04-19
status: refined
tags: [github, auto-close, commit-message, pr, parsing, syntax]
related:
  - ../meta/patterns/sub-agent-finalization-miss-pattern.md
---

## 요약

PR/커밋 메시지에서 `Closes: #105, #110` (콜론 사용 + 콤마로 여러 이슈) 문법은 **첫 번째 이슈만 auto-close** 되고 두 번째 이후는 파싱 실패한다. **각 이슈마다 별도 keyword 사용 필수**.

## 본문

#### 관찰 사례

harness-setting PR #108 (v2.14.0) 커밋 메시지:

```
Closes: #105, #110
```

결과: **#105 만 auto-close, #110 은 OPEN 유지**. 수동 `gh issue close 110` 필요.

#### GitHub 공식 지원 closing keyword

- `close`, `closes`, `closed`
- `fix`, `fixes`, `fixed`
- `resolve`, `resolves`, `resolved`

#### 올바른 문법 (여러 이슈)

```
Closes #105, closes #110              # ✅ 둘 다 인식
Closes #105
Closes #110                            # ✅ 별도 줄로 분리
Fixes #105, fixes #110                 # ✅
```

#### 인식 실패 문법

```
Closes: #105, #110                     # ❌ 콜론 + 콤마. #105 만
Closes #105, #110                      # ❌ 콤마만 있고 두번째에 keyword 없음
Closes #105 #110                       # ❌ keyword 없는 두번째
```

#### 실측 근거

- 파싱 결과 비교: GitHub 은 각 이슈마다 바로 앞 단어에 keyword 가 있어야 인식
- 콜론 (`:`) 는 keyword 와 이슈 번호 사이 허용되지 않는 구분자

## 교훈

1. **PR 설명/커밋 메시지에 여러 이슈 close 시 각각 keyword 반복** — `Closes #A, closes #B` 또는 줄 분리
2. **자동화 도구 (create-pr 스킬 등) 의 템플릿에 올바른 문법 박제** — 잘못된 선례가 템플릿화되면 반복됨
3. **PR merge 직후 이슈 상태 확인 루틴** — auto-close 실패를 구조적으로 감지. `gh issue view <번호> --json state` 로 close 확인. sub-agent 마무리 체크리스트 (volt #24) 의 일부로 통합 가능
4. **v2.14.0 이후 harness-setting 의 PR 템플릿** 에 여러 이슈 closing 예시 명시 예정

## 관련 노트/링크

- GitHub Docs: https://docs.github.com/en/get-started/writing-on-github/working-with-advanced-formatting/using-keywords-in-issues-and-pull-requests
- harness-setting PR #108 (실제 발생 사례)
- volt #24 (sub-agent 마무리 보고 누락 4회) 의 auto-close 누락 버전
