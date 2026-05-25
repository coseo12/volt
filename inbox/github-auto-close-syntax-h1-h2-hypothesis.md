---
title: GitHub auto-close 비결정 가설 H1/H2 — `## Closes` H2 헤더 간섭 + `- Closes #N (부연)` 불릿+괄호 조합
type: knowledge
source_repo: coseo12/harness-setting
source_issue: 65
captured_at: 2026-04-22
status: inbox
tags: [github, auto-close, closing-keyword, syntax, parsing, h1-hypothesis, h2-hypothesis, statistical-baseline]
related:
  - ./github-auto-close-keyword-syntax.md
  - ./closing-refs-base-develop-trap.md
  - ./closing-refs-base-develop-32nd-reproduction.md
---

## 요약

GitHub auto-close 가 **같은 PR 본문 / 동일 토큰 `Closes #N`** 인데 비결정적으로 인식했다. 두 syntax-sensitive 가설 박제:

- **H1**: `## Closes` H2 헤더 + 다음 줄 `Closes #N` 배치 → 헤더의 "Closes" 가 keyword 로 먼저 소비되며 다음 줄 미인식 (1회 관찰)
- **H2**: `- Closes #N (부연)` 불릿+괄호 조합 → 파서가 괄호 앞까지만 토큰 처리 후 match 실패 (1회 관찰)

반증 데이터: 단독 라인 `Closes #N` 패턴은 harness-setting 11/11 (100%) 인식 — H3(서버 비결정) 가설 약화.

본 가설들은 **별도 차원**: 후속 #115/#117 에서 확정된 base=develop 함정과 메커니즘 분리. base=default(main) 시에도 syntax 만으로 미인식 가능성이 남는다.

## 본문

### 1차 관찰 — harness-setting #210 release PR (2026-04-22)

`coseo12/harness-setting` release PR [#210](https://github.com/coseo12/harness-setting/pull/210) (develop → main, merge commit) 머지 시점에 동일 PR 본문 내 4건의 `Closes #N` 중 **2건만 인식**:

| feature PR | 본문 keyword | 대상 이슈 | 결과 |
|---|---|---|---|
| #206 | `Closes #196` (단독 줄) | #196 | **CLOSED** ✓ |
| #205 | `Closes #202` (단독 줄) | #202 | **CLOSED** ✓ |
| #198 | `## Closes\nCloses #197` (H2 헤더 + 다음 줄) | #197 | **OPEN** ✗ |
| #201 | `## Closes\nCloses #200` (H2 헤더 + 다음 줄) | #200 | **OPEN** ✗ |
| #204 | `Refs #199` (의도적) | #199 | OPEN (의도) |

→ **H1 가설** 도출: H2 헤더의 "Closes" 가 keyword 로 먼저 소비되어 바로 다음 줄의 `Closes #N` 이 무시될 가능성.

부수 관찰: CLOSED 된 #196 의 `closedByPullRequestsReferences` 필드도 빈 배열 — GitHub API 가 auto-close 의 PR linkage 를 일관되게 기록하지 않음.

### 2차 관찰 — astro-simulator #291 (2026-04-22, 같은 날)

`coseo12/astro-simulator` PR [#291](https://github.com/coseo12/astro-simulator/pull/291) (feature → develop, **squash merge**) 미작동 vs PR [#293](https://github.com/coseo12/astro-simulator/pull/293) (squash merge, base=develop) 정상:

| PR | 머지 | Closes 문법 | 결과 |
|---|---|---|---|
| #291 | squash | `## Closes\n- Closes #271 (P10 known issue — ...)\n- Closes #292 (...)` | **실패** #271/#292 OPEN |
| #293 | squash | `Closes #225` (단순 라인) | **성공** #225 CLOSED |

→ **H2 가설** 도출: 불릿 `- ` 접두 + 괄호 `(...)` 부연 조합이 파서에서 토큰 매칭 실패. `Closes #N (...)` 에서 괄호 바로 앞까지 파서가 토큰을 삼키고 뒤에 오는 내용 때문에 match 실패 추정 (미검증).

(주의: 본 시점은 base=develop 함정 발견 이전. PR #291 의 미작동은 사후적으로 H2 + base 함정 **양쪽** 으로 설명 가능 — 단독 라인 PR #293 은 base=develop 인데도 작동했으므로 base 함정 단독으로는 설명 불가. H2 가설 잔존.)

### 반증 데이터 — harness-setting 단독라인 전수 조사 (2026-04-25)

같은 레포 최근 8 PR / 11 이슈 (단독 라인 패턴):

| PR | Closes 패턴 | 결과 |
|---|---|---|
| #227 | `Closes #208` | CLOSED ✓ |
| #223 | `Closes #190` | CLOSED ✓ |
| #220 | `Closes #184` | CLOSED ✓ |
| #217 | `Closes #203` | CLOSED ✓ |
| #216 | `Closes #213` | CLOSED ✓ |
| #212 | `Closes #195\nCloses #194\nCloses #199` (줄 분리, 3건) | 3/3 CLOSED ✓ |
| #211 | `Closes #207` | CLOSED ✓ |
| #231 | `Closes #228` | CLOSED ✓ |

→ **단독 라인 11/11 (100%) 인식**. H3 (GitHub 서버 단순 비결정 버그) 가설 **약화**.

### 가설 추적 카운트

| 가설 | 미인식 사례 | 재현 카운트 |
|---|---|---|
| **H1**: `## Closes` H2 헤더 + 다음 줄 keyword | harness #198/#201 (1 PR, 2 이슈) | **1 PR** (3회 미달) |
| **H2**: `- Closes #N (부연)` 불릿+괄호 | astro-simulator #271/#292 (1 PR, 2 이슈) | **1 PR** (3회 미달) |
| **H3**: GitHub 서버 비결정 | — | **약화** (11/11 인식) |

> PR 1건 내부에서 동일 패턴이 N건 미인식되는 것은 **단일 인스턴스** 로 카운트 (하나의 파서 입력 = 하나의 비결정 사건).

### 후속 메커니즘 분리 (참조)

본 가설 박제 이후 #115/#117 에서 **base=develop 함정** (default branch 외 머지 시 closing keyword 100% 미발화) 이 통계 11/11 로 확정. 그러나 H1/H2 는 base 함정과 **독립 차원** — base=default(main) 시에도 syntax 결함으로 미인식 가능성 잔존. 후속 재현 시 base 조건과 syntax 조건을 분리 측정 필요.

### 검증 방법 제안 (재현 트리거)

의도적으로 같은 base=main PR 본문에 **3개 패턴 병기** 후 state 실측:

1. `Closes #A` (단독 라인 — 정상 control)
2. `## Closes\nCloses #B` (H1 검증)
3. `- Closes #C (부연)` (H2 검증)

## 교훈

1. **단독 라인 (`Closes #N`) 패턴은 신뢰 가능** — 반증 0건/11. PR 본문/커밋 메시지 표준으로 강제 가치.
2. **H1/H2 두 가설은 본문 박제 기준 (3회 재현) 미달** — 추적 계속, 누적 시 CLAUDE.md `## PR 규칙` 에 가드 추가 검토.
3. **`closedByPullRequestsReferences` 빈 배열은 부수 결함** — auto-close 가 작동해도 GitHub API 가 PR linkage 를 일관되게 기록하지 않음. 마무리 체크리스트가 PR 본문/커밋을 SSoT 로 보강해야 한다.
4. **메인 오케스트레이터 마무리 루틴** — `gh issue view <N> --json state` 독립 검증으로 auto-close 실패 구조적 감지 (volt #24 sub-agent SSoT 의 메인 확장).
5. **release PR 본문에 명시 나열** — `Closes #A\nCloses #B\n...` 형식. 누적 카운트 도달 시 CLAUDE.md 규약화 후보.

## 관련 노트/링크

- 본 이슈 원본 (선행 사례): [volt #41](https://github.com/coseo12/volt/issues/41) — `Closes: #A, #B` 콜론 문법 미인식 (문법 명확한 결함)
- 후속 메커니즘 분리: [volt #115](https://github.com/coseo12/volt/issues/115) (base=develop 함정 100% 미발화 통계), [volt #117](https://github.com/coseo12/volt/issues/117) (재현 +3건 누적)
- sub-agent 박제 누락 전반: [volt #24](https://github.com/coseo12/volt/issues/24)
- GitHub Docs: https://docs.github.com/en/get-started/writing-on-github/working-with-advanced-formatting/using-keywords-in-issues-and-pull-requests
- 실제 발생 PR: harness-setting #210/#198/#201, astro-simulator #291
