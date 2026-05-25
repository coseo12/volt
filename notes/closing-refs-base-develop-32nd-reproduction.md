---
title: closingIssuesReferences=[] 함정 32회차 — base=develop 변형 재현 누적 (#115 가설 강화)
type: knowledge
source_repo: coseo12/astro-simulator
source_issue: 117
captured_at: 2026-05-25
status: refined
tags: [github, auto-close, closing-keyword, base-develop, default-branch, statistical-confirmation, anti-pattern]
related:
  - ./closing-refs-base-develop-trap.md
  - ../meta/patterns/closing-issues-references-empirical-verify.md
---

## 요약

volt [#115](https://github.com/coseo12/volt/issues/115) 가설 (base=develop PR 머지 시 GitHub closing keyword 100% 미발화) 의 본 세션 재현 누적. base=develop PR 의 `Closes #N` 콜론 없는 정확 문법인데도 `closingIssuesReferences=[]` 발생 — 본 세션만 **3건 누적 재현**. 매번 수동 close 의무.

## 본문

**본 세션 재현 누적 (astro-simulator, 2026-05-17 ~ 2026-05-18)**:

| PR | Closes 키워드 | base | closingIssuesReferences | 수동 close 의무 |
|----|---------------|------|--------------------------|-----------------|
| #496 (#480 SSoT drift 자동화) | `Closes #480` | develop | **[]** | ✓ |
| #497 (#473 PR 템플릿 backstop) | `Closes #473` | develop | **[]** | ✓ |
| #498 (#495 Amendment 4) | `Closes #495` | develop | **[]** | ✓ |

**누적 통계** (volt #115 + 본 재현):

| 데이터셋 | base=develop PR | keyword 보유 | closing_refs 작동 |
|----------|-----------------|--------------|-------------------|
| volt #115 (2026-05-17 박제) | 10건 | 8건 | **0건** (0%) |
| 본 세션 (2026-05-18) | 3건 | 3건 | **0건** (0%) |
| **합계** | **13건** | **11건** | **0건** (0%) |

11/11 = **100% 미발화**. volt #115 가설 통계 강화. GitHub 공식 동작 확정.

**메인 오케스트레이터 새 의무 (확정)**:

base=develop PR 머지 후 **무조건 수동 close** + 코멘트에 박제 (cross-link + 함정 32회차 박제 누적):

```bash
gh issue close <N> --reason completed --comment "...

## auto-close 미발화 (volt #115 base 함정 32회차 재현)
PR base=develop → closingIssuesReferences=[]. Closes #N 콜론 없는 정확 문법이어도 미발화.
gitflow base=develop 운영 시 GitHub closing keyword 100% 무효 (volt #115 통계로 확정).
수동 close.
"
```

**왜 함정 32 회차인가**:

- CLAUDE.md memory 박제 누적: "closingIssuesReferences=[] 함정 31회차" → 본 세션 재현 +3 = **32~34 회차** (Z 패턴 카운트 보다 더 빈도 높은 함정)
- 회차 카운트 자체는 메모리 휘발성 (volt #113 keyword-only 박제 한계) — 통계 누적이 더 신뢰
- 함정 A (콜론) vs 함정 B (base) 메커니즘 분리 후 본 세션 3건 모두 함정 B (base=develop)

**일반화 — gitflow 운영 시 자동화 정책**:

1. PR 템플릿 본문에 `Closes #N` 박제 의무 (다음 release PR 의 main 흘러들어가는 가능성 + 수동 close 코멘트 cross-link 정합)
2. 메인 오케스트레이터 마무리 체크리스트 의무 — `auto_close_issue_states` field 가 SSoT (메인이 `gh issue view <N> --json state` 로 독립 확인)
3. CI workflow 도입 후보 — PR 머지 후 `Closes #N` 키워드 자동 발화 (단, base 함정 인지 + 수동 close 대체)

## 관련 노트/링크

- 가설 박제: volt [#115](https://github.com/coseo12/volt/issues/115) (base=develop 함정 100% 미발화 통계 증거)
- 함정 A (콜론 문법) 분리: volt [#93](https://github.com/coseo12/volt/issues/93)
- 본 세션 재현: astro-simulator PR #496, #497, #498 (모두 base=develop, 모두 수동 close)
- 누적 메모리 박제: "closingIssuesReferences=[] 함정 31회차" → 본 변형으로 32~34 회차 (메모리 회차 카운트는 휘발성, 본 통계가 SSoT)
