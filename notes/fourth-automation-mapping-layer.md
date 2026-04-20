---
title: 3층 방어 완성 후 4번째 "자동 매핑" 층의 필요성 — 수동 연결점은 마지막까지 남아있다
type: knowledge
source_repo: coseo12/harness-setting (v2.16.0 → v2.17.0 → v2.18.0 → v2.19.0 의 4단계 방어 완성 여정)
source_issue: 43
captured_at: 2026-04-19
status: refined
tags: [defense-layers, wishful-documentation, structural-enforcement, automation, agent-prompt, shell-script, json-outcome-file]
related:
  - ./state-atomicity-multi-layer-defense.md
  - ../meta/patterns/cross-validate-after-policy-freeze.md
  - ./gemini-capacity-fallback-retry-protocol.md
  - ./sub-agent-runtime-ssot-variance.md
---

## 요약

원래 "3층 방어" (선언 + 프롬프트 + 스크립트) 로 충분해 보이지만 **실제로는 4번째 "자동 매핑" 층이 수동 연결점을 제거한다**. 스크립트가 exit code 77 을 반환해도 에이전트가 "이것을 어떻게 해석할지" 는 여전히 수동 판단이 필요하다. outcome JSON 파일 + 공통 스키마 파싱 스니펫이 마지막 판정 자의성을 거세한다. harness-setting 의 매니페스트 원자성 3계층 (volt [#28](https://github.com/coseo12/volt/issues/28)) 과 대비되는 패턴 — 같은 "계층 방어" 사고틀에서도 **자동화 포인트가 다르다**.

## 본문

#### 3층 방어 (선언 / 프롬프트 / 스크립트) 만으로 부족한 이유

harness 의 cross-validate 폴백 프로토콜 완성 여정:

| 릴리스 | 층 | 내용 | 남은 수동 연결점 |
|---|---|---|---|
| v2.16.0 | **선언** | CLAUDE.md `## 교차검증` 3단계 명시 | 프롬프트가 규약을 "따를 것" 기대 (Gemini 가 "wishful documentation" 으로 지적) |
| v2.17.0 | **프롬프트** | 5개 에이전트 공통 JSON 스키마 하드코딩 | 스크립트가 실제로 규약대로 동작하는지 보장 없음 |
| v2.18.0 | **스크립트** | `cross_validate.sh` 429 폴백 + exit 77 + stderr 프리픽스 | **에이전트가 exit 77 을 "어떻게 해석할지" 는 여전히 수동 판단** |
| v2.19.0 | **자동 매핑** | outcome JSON 파일 + architect bash 스니펫 자동 파싱 | (없음 — 1:1 복사만 남음) |

v2.18.0 에서 "3층 방어 완성" 이라 선언했지만, PR #130 의 Gemini cross-validate 에서 "에이전트가 스크립트 exit code 를 **주관적으로 해석** 하는 여지가 여전히 남아있다" 고 지적받음 (암묵적). v2.19.0 의 Phase 3 이 이를 outcome JSON 으로 해소.

#### 수동 연결점의 3가지 형태

| 형태 | 예시 | 제거 방법 |
|---|---|---|
| **의미 해석** | exit 77 을 "Gemini capacity 폴백" 으로 이해 | 구조화된 outcome 필드 (`"429-fallback-claude-only"`) |
| **경로 추적** | 최신 로그 파일 찾기 (`ls -t ...`) | stdout 에 명시 prefix (`[outcome-file] <경로>`) 출력 |
| **값 매핑** | outcome → `extends.cross_validate_outcome` | 공통 JSON 스키마 + 1:1 복사 스니펫 |

3층 방어는 **의미 해석 수동 연결점** 만 남긴다. 자동 매핑 층은 그 마지막 연결점을 제거.

#### 매니페스트 원자성 3계층 (volt [#28](https://github.com/coseo12/volt/issues/28)) 과의 차이

매니페스트 원자성은 3계층 (도중 / 사후 / 안내) 에서 **사용자** 가 복구 action 을 취하는 수동 연결점이 의도적으로 남아있다. "사용자가 `--apply-all-safe` 를 실행해야 계층 2 가 활성화" 가 계층 3 의 존재 근거. 사용자 개입 자체가 시스템 디자인의 일부.

반면 cross-validate 폴백은 **에이전트 자동 실행** 을 목표 — 사용자 개입이 없는 것이 이상적. 따라서 "에이전트의 주관적 판단" 이라는 수동 연결점도 제거해야 한다. 4번째 자동 매핑 층이 필요한 이유.

#### 일반화 — 언제 4번째 층이 필요한가?

- **에이전트/자동화 주체** 가 의미 해석을 수행해야 하는 경우 (사용자 개입 불가)
- 해석 결과가 **다음 행동** (JSON 필드 매핑 / 조건 분기 / 상태 전파) 에 직접 영향
- 해석 자의성이 결정론적 재현성을 해치는 경우

반대로 **사용자가 주체** 인 경우 (매니페스트 복구, 에러 리포트 확인) 는 3층이면 충분. 안내(계층 3) 로 끝난다.

#### 설계 체크리스트

계층 방어 설계 시 "해석자가 누구인가" 를 묻기:
- 인간 (운영자 / 개발자) → 3층 (도중 / 사후 / 안내) 으로 충분
- 자동화 주체 (에이전트 / CI / 스크립트) → **4층** (+ 자동 매핑) 필요
- 혼합 → 주 해석자 기준 + 보조 해석자용 병렬 경로

## 관련 노트/링크

- volt [#28](https://github.com/coseo12/volt/issues/28) — 매니페스트 원자성 3계층 방어 (대비 사례, 사용자 주체)
- volt [#23](https://github.com/coseo12/volt/issues/23) / [#40](https://github.com/coseo12/volt/issues/40) — 박제 직후 cross-validate 루틴 + 429 폴백 프로토콜
- harness [#119](https://github.com/coseo12/harness-setting/issues/119) Phase 1 + 2 + [#131](https://github.com/coseo12/harness-setting/issues/131) Phase 3 — 4층 완성 경로
- harness releases [v2.16.0](https://github.com/coseo12/harness-setting/releases/tag/v2.16.0) ~ [v2.19.0](https://github.com/coseo12/harness-setting/releases/tag/v2.19.0)
- harness `docs/architecture/state-atomicity-3-layer-defense.md` — 3계층 방어 일반화 문서 (이것과 대비 관점 추가 가치)
