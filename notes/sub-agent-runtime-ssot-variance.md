---
title: sub-agent 반환 JSON SSoT variance — 정적 파일 SSoT ≠ 런타임 반환 SSoT 의 직교성 (3회 관찰)
type: knowledge
source_repo: coseo12/harness-setting
source_issue: 58
captured_at: 2026-04-20
status: refined
tags: [sub-agent, ssot, llm-variance, runtime-validation, static-vs-runtime-blindspot, return-json, compile-rule-activation]
related:
  - ../meta/patterns/sub-agent-finalization-miss-pattern.md
  - ./fourth-automation-mapping-layer.md
  - ./cross-validate-principle-declaration-bias.md
  - ./release-version-metadata-drift-guard.md
  - ./ci-multi-language-design-4-principles.md
---

## 요약

**정적 파일 검증** 과 **런타임 반환값 검증** 은 서로 다른 층이며, 둘 중 하나만으로는 SSoT 강제가 성립하지 않는다. harness-setting v2.26.0 에서 공통 JSON 스키마를 9필드로 확장한 후, `scripts/verify-agent-ssot.sh` 가 파일 차원 정합성 (5 files × 9 fields = 45 checks) 은 완벽히 보장하지만, **sub-agent 가 런타임에 실제로 반환하는 JSON** 은 3회 연속 규약 이탈. LLM 기반 sub-agent 의 "system prompt 최신 박제를 반환값에 일관되게 반영하지 못하는 variance" 라는 **구조적 한계** 가 정적 검증의 blindspot 으로 드러난 사례. knowledge-compilation "3회 이상 관찰 시 박제" 규약 발동 조건 달성.

## 본문

#### 관찰 사례 3건 (2026-04-20 단일 세션)

| PR | 릴리스 | sub-agent | 신규 2필드 반환 상태 | variance 유형 |
|---|---|---|---|---|
| [harness#167](https://github.com/coseo12/harness-setting/pull/167) | v2.27.0 | reviewer | `spawned_bg_pids` / `bg_process_handoff` **필드 자체 누락** | **유형 1: 완전 누락** |
| [harness#170](https://github.com/coseo12/harness-setting/pull/170) | v2.28.0 | reviewer | `null` / `null` | **유형 2: 기본값 이탈** (파일 규약 `[]` + `"none"` 과 다름, CLAUDE.md 규약상 `null` 은 허용 — 스키마 위반 아님) |
| [harness#178](https://github.com/coseo12/harness-setting/pull/178) | v2.29.0 | reviewer | **필드 자체 누락** (#167 패턴 재현) | **유형 1 재발** |

3 회 모두 `.claude/agents/reviewer.md` 파일 자체에는 9필드 전부 올바르게 기재 (verify-agent-ssot.sh 45/45 통과). **파일 정적 상태** 와 **sub-agent 런타임 반환** 사이의 괴리.

#### 정적 vs 런타임 SSoT 검증의 직교성

| 층 | 대상 | 검증 시점 | 본 세션 가드 | 커버 | blindspot |
|---|---|---|---|---|---|
| **정적** | `.claude/agents/*.md` 파일의 JSON 블록 | PR 머지 전 CI | `verify-agent-ssot.sh` (v2.23.0, 9필드 확장 v2.26.0) | 파일 drift 100% | 런타임 반환값 |
| **런타임** | sub-agent 가 실제 반환한 JSON | sub-agent 종료 직후 | **(없음)** | — | LLM variance 로 필드 생략 / `null` 대체 / 기본값 이탈 |

정적 가드는 "작성자가 파일을 올바르게 수정했는가" 를 검증. 런타임 가드는 "sub-agent 가 파일 규약대로 동작했는가" 를 검증. 둘은 **직교** — 한쪽만으로 충분하지 않다.

#### 구조적 원인 — LLM sub-agent 의 프롬프트 반영 variance

1. **시스템 프롬프트 컨텍스트 길이 증가 시 후반부 지시 반영도 감소** — v2.26.0 에서 2필드 추가하면서 전체 체크리스트 JSON 블록이 길어짐. LLM 이 마지막 2필드를 "덜 중요하게" 처리할 가능성
2. **기본값 vs null 선택의 자의성** — 규약이 "누락 field 는 `null` 또는 빈 배열/객체 명시" 로 둘 다 허용. LLM 이 매번 다른 선택 (유형 1 vs 유형 2)
3. **training data bias** — "표준 JSON 반환" 이라는 태스크에서 LLM 은 최소 필드만 반환하는 편향. extends 영역이 충분히 채워지면 코어 9필드 중 일부를 생략

#### 3층 방어 적용 가능성 (volt #43 / state-atomicity pattern)

본 문제에 대해 volt #43 의 **"해석자가 자동화 주체일 때 4번째 자동 매핑 층" 패턴** 이 적용 가능:

| 계층 | 내용 (본 variance 에 매핑) | 이미 존재? |
|---|---|---|
| 1. 선언 | CLAUDE.md `### sub-agent 검증 완료` SSoT 블록 | ✅ (v2.26.0) |
| 2. 프롬프트 | 5 에이전트 파일의 체크리스트 JSON 블록 (9필드) | ✅ (v2.26.0) |
| 3. 정적 스크립트 | `verify-agent-ssot.sh` 파일 drift 차단 | ✅ (v2.23.0) |
| **4. 런타임 매핑** | sub-agent 반환 JSON 의 필드 존재 + 타입 검증 | ❌ **현재 없음** |

즉 v2.26.0 기준으로는 "3층 방어 선언" 이지만 cross-validate 폴백 프로토콜 (v2.21.0 4층 완성) 과 달리 **4층이 비어 있다**. 수동 연결점: "메인 오케스트레이터가 sub-agent 반환을 주관적으로 검토" — variance 자동 감지 없음.

#### 해결 후보 4안 (brainstorm)

**A. 메인 post-parse 가드**: Agent 반환 직후 JSON 파싱 → 9필드 존재·타입 1차 검증 → 누락 경고
- 장점: 즉시 적용
- 단점: 사후 점검, sub-agent 수정 아님

**B. JSON Schema + 공통 헬퍼 스크립트**: `.claude/agents/schema/checklist.schema.json` + `scripts/verify-agent-return-schema.sh`
- 장점: 엄격 검증 + extends 자유도
- 단점: jq / Node 의존 (volt #51 "외부 툴 주장 실측" — jq 도입 시 parse-cross-validate-outcome NO-OP 결정과 상충 여부 확인 필요)

**C. 에이전트 프롬프트 자가 검증 절차**: 각 에이전트에 "JSON 반환 직전 9필드 자가 체크" 추가
- 장점: 인프라 불필요
- 단점: **variance 가 이 문제의 원인** — 자가 검증도 variance 에 노출 (메타 순환)

**D. 복합 3층 (A + B + C)**: LLM 자가 체크 → 메인 post-parse → schema 검증 스크립트
- volt #43 3층 방어 패턴의 직접 적용. 각 층이 직교 타이밍 커버

#### 일반화 — "정적 파일 SSoT ≠ 런타임 반환 SSoT"

적용 가능 시나리오:
- **API contract testing** — OpenAPI spec (정적) vs 실 서버 응답 (런타임) — Pact / Dredd 류
- **DB migration + 런타임 ORM** — 스키마 파일 (정적) vs ORM 이 실제 쿼리 결과로 받는 컬럼 (런타임)
- **타입 선언 vs 런타임 값** — TypeScript `.d.ts` (정적) vs 실행 시 `typeof` / io-ts / zod 검증 (런타임)
- **환경 변수 스키마 vs 프로세스 env** — dotenv.example (정적) vs `process.env` 런타임 감지
- **LLM sub-agent system prompt vs 반환값** — 본 사례

공통 조건:
- 두 층이 **동일한 계약** 을 표현해야 함
- 정적 층은 작성자 실수를, 런타임 층은 동작 실수를 각각 탐지
- 둘 중 하나라도 blindspot 있으면 드리프트 발생 (본 사례 3회 관찰)

## 교훈 / 다음에 적용할 점

1. **"정적 SSoT 가드 작동 ≠ 런타임 동작 일치"** 를 규칙으로 박제 — 정적 스크립트 통과를 신뢰 끝점으로 쓰면 안 됨
2. **신규 필드 추가 PR 직후 variance 고위험** — 본 사례는 v2.26.0 (2필드 추가) 직후 3회 연속 관찰. 추가 직후가 LLM 프롬프트 반영 지연 고위험 구간
3. **메인 오케스트레이터의 post-agent 검증 책임 박제** — CLAUDE.md `### sub-agent 검증 완료 ≠ GitHub 박제 완료` 의 "감점 처리" 규약을 수동 관찰이 아닌 **구조적 자동 감지** 로 승격 필요 (volt #56 "암묵 관례의 구조적 검증 승격" 과 동일 패턴)
4. **3층 방어 패턴 재적용** — 정책/설계 박제의 3층 방어 (선언 + 프롬프트 + 스크립트) 가 sub-agent 반환에서 **4층 (런타임 매핑)** 이 빠져있음. volt #43 의 직접 재사용

## 관련 노트/링크

- 관찰 PR: [harness#167](https://github.com/coseo12/harness-setting/pull/167), [harness#170](https://github.com/coseo12/harness-setting/pull/170), [harness#178](https://github.com/coseo12/harness-setting/pull/178)
- 선행 volt [#24](https://github.com/coseo12/volt/issues/24) (sub-agent 마무리 보고 누락), [#43](https://github.com/coseo12/volt/issues/43) (4번째 자동 매핑 층), [#55](https://github.com/coseo12/volt/issues/55) (원칙 선언 후 cross-validate), [#56](https://github.com/coseo12/volt/issues/56) (암묵 관례 구조적 승격), [#57](https://github.com/coseo12/volt/issues/57) (다언어 CI 4원칙)
- harness 후속 이슈: **본 관찰의 구조적 해결** 은 harness 에서 별도 이슈로 분리 (실행 가능 문제 정의)
