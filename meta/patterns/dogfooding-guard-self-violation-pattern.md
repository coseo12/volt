---
title: volt #51 가드 셀프 위반 — pnpm --if-present flag 호환 암묵 가정 (dogfooding 실패 1회 즉시 박제)
type: pattern
source_repo: coseo12/harness-setting
source_issue: 59
captured_at: 2026-04-20
status: refined
tags: [cross-validate, dogfooding-failure, external-tool-verification, pnpm, npm-flag-asymmetry, self-violation, compile-rule-exception, reviewer-gemini-both-missed]
related:
  - ../../notes/external-tool-claim-empirical-verify.md
  - ../../notes/cross-validate-principle-declaration-bias.md
  - ../../notes/downstream-measurement-final-guard.md
  - ../../notes/claude-bias-5th-ecosystem-assumption.md
---

## 배경/상황

harness-setting v2.26.0 에서 volt [#51](https://github.com/coseo12/volt/issues/51) 교훈을 박제한 `## 교차검증` 섹션의 **"외부 툴 동작 주장은 실측 필수"** 가드 (검증 필수도 매트릭스 "외부 툴 동작 / CI / 프레임워크 기본값" 카테고리 최고 등급). 본 가드가 **박제 이후 Claude 자신에 의해 위반** 된 사례가 2026-04-20 세션에서 1회 관찰 — 방어 가드의 **셀프 위반 (dogfooding 실패)** 패턴.

## 내용

#### 실패 사례 상세

**v2.28.2 (PR [harness#176](https://github.com/coseo12/harness-setting/pull/176))** 에서 CI `detect-and-test` pnpm 경로 도입 시 Claude 는 기존 npm step 의 `--if-present` 패턴을 **실측 없이** pnpm 에 복사:

```yaml
# npm (올바름 — --if-present 가 npm 네이티브 플래그)
- name: npm test
  if: hashFiles('package.json') != ''
  run: npm test --if-present

# pnpm (v2.28.2 실패 — --if-present 가 pnpm 플래그 아님, script args 로 forward)
- name: pnpm test
  if: hashFiles('pnpm-lock.yaml') != ''
  run: pnpm test --if-present
```

**실 동작**: pnpm 은 `--if-present` 를 자신의 플래그로 인식하지 못하고 `pnpm -r test --if-present` 형태로 workspace 각 패키지의 script args 로 **forward**. vitest/jest 등 런너가 `CACError: Unknown option "--ifPresent"` 로 실패.

**올바른 사용법**: `pnpm run --if-present test` (pnpm 8+ 공식 지원) — `--if-present` 를 script 이름 **앞** 에 배치해야 pnpm 이 자체 플래그로 인식.

#### 감지 체인 타임라인

1. **v2.28.2 릴리스 (2026-04-20T10:37)** — 내가 직접 pnpm 경로 구현, 실측 없이 npm 패턴 이식
2. **v2.29.0 릴리스 (2026-04-20T10:50)** — 다언어 CI 확장에서 동일 pnpm step 유지. **Reviewer + Gemini cross-validate 모두 미탐지**
3. **다운스트림 astro-simulator#270** 에서 v2.28.2 적용 시 `detect-and-test` **red 로 최초 발견**
4. **issue [harness#181](https://github.com/coseo12/harness-setting/issues/181)** 보고 + **PR [harness#182](https://github.com/coseo12/harness-setting/pull/182)** fix → v2.29.1 PATCH 릴리스 (2026-04-20T10:56)

**총 지연**: 박제 후 첫 실측 실패까지 **19분** — 빠른 발견이나 2 릴리스 (v2.28.2 / v2.29.0) 다운스트림 영향 후 발견.

#### 왜 가드가 자기 자신을 못 막았나 — 3가지 구조적 원인

1. **"같은 생태계 도구 = 동일 플래그" 의 암묵 전제** — npm/pnpm/yarn 이 모두 Node.js 생태계라는 이유로 flag 호환성을 암묵 가정. volt #51 가드의 매트릭스에 "같은 카테고리 내 도구는 CLI 호환 가정 금지" 가 명시되지 않음
2. **cross-validate 가 동일 편향을 공유** — Gemini 도 "npm 과 pnpm 은 비슷하니 `--if-present` 동일 동작" 편향에서 자유롭지 않음. volt #51 케이스 A (setup-node cache 주장 실측 반증) 는 Gemini **주장 실측** 이었는데, 본 사례는 Gemini **미언급**. 즉 Gemini 는 주장해야 반증 가능하고, 미언급하면 가드가 발동 안 함
3. **Reviewer 도 "실측 자료 없음" 상태에서는 무력** — reviewer sub-agent 도 diff 만 보고 패턴의 타당성을 판단. "이 flag 가 실제 pnpm 에서 어떻게 동작하는지" 실측 없이는 구조적 차이 감지 불가

#### dogfooding 성공/실패의 비대칭성

| 관찰 유형 | 본 세션 누적 횟수 | 박제 효과 |
|---|---|---|
| 정상 경로 성공 | **9회** (volt #42 / #44 / 본 세션 PR #113/#117/#126/#130/#134/#164/#170/#178 cross-validate) | 축적 — knowledge-compilation 규약 "3회 이상" 기준 충족 |
| 셀프 위반 (가드 실패) | **1회** (본 사례 pnpm `--if-present`) | **박제 가치 즉시 충분** — 1회도 drop 금지 |

**비대칭 이유**: 성공은 "시스템이 정상 작동" 을 누적 증명하나, 실패는 **가드의 경계선을 단 한 번에 드러냄**. compile 규약 "3회 이상" 은 **관찰 분산 패턴** 에 적용되고, **가드 자기 위반은 즉시 박제** 예외.

#### volt #51 가드의 2차 방어선 제안

**보완안**: 가드 문구에 다음 2개 절차 추가:

1. **"외부 툴 flag 는 공식 문서 (CLI help / 공식 README) 에서 직접 확인 후 인용"** — LLM 지식 (training cutoff) 이 아닌 실시간 문서
2. **"같은 생태계 내 도구 간 flag 호환 가정 금지"** — npm/pnpm/yarn/bun 가 같은 Node.js 생태계여도 CLI flag 는 독립 설계. 반드시 각 도구의 플래그 네이티브 지원 여부 개별 확인

추가 실행 절차: cross-validate 호출 프롬프트에 **"이 PR 은 `<도구>` 의 어떤 flag 를 사용하는가? 각 flag 가 해당 도구의 네이티브 지원 flag 인지 공식 문서로 확인"** 질문 템플릿 삽입.

## 교훈 / 다음에 적용할 점

1. **가드 박제 ≠ 가드 영속 동작** — 박제 후에도 같은 Claude 가 같은 편향으로 위반 가능. 박제는 알림이지 강제가 아님
2. **"성공 관찰 축적 vs 실패 관찰 즉시 박제" 비대칭** 을 compile 규약에 명시 — 본 이슈가 "1회 관찰" 이지만 규약 예외 근거 박제
3. **cross-validate 의 blindspot** — Gemini 가 제안하지 않으면 가드 발동 안 함. "Gemini 의 침묵" 도 detection 대상에 포함해야 (예: "본 PR 에서 새로 도입한 외부 도구가 있는가? 각 도구의 주요 flag 가 공식 문서와 일치하는가?" 명시 질문)
4. **생태계 경계 ≠ flag 경계** — Node.js 생태계 전반에서 `--if-present` 처럼 npm-특정 플래그를 다른 도구로 이식 시 반드시 실측. yarn/bun/deno/pnpm 각각 독립 CLI 설계
5. **다운스트림 실측이 최종 가드** — upstream 의 단위 테스트 / reviewer / cross-validate 모두 미탐지 상태에서 astro-simulator 의 실 CI red 로 최초 감지. "실 프로덕션 사용" 이 여전히 최후의 방어선

## 관련 노트/링크

- 실패 원본 PR: [harness#176](https://github.com/coseo12/harness-setting/pull/176) (v2.28.2 pnpm 경로 도입)
- 실패 지속 PR: [harness#178](https://github.com/coseo12/harness-setting/pull/178) (v2.29.0 — 본 세션에서 검증했으나 미탐지)
- 수정 체인: [harness#181](https://github.com/coseo12/harness-setting/issues/181), [harness#182](https://github.com/coseo12/harness-setting/pull/182), [harness#183](https://github.com/coseo12/harness-setting/pull/183)
- 다운스트림 최초 감지: astro-simulator#270
- 선행 volt [#51](https://github.com/coseo12/volt/issues/51) (본 가드의 원형), [#42](https://github.com/coseo12/volt/issues/42) / [#44](https://github.com/coseo12/volt/issues/44) (정상 경로 성공 축적의 대조 사례), [#55](https://github.com/coseo12/volt/issues/55) (Claude 편향 4종 — 본 사례는 5번째 편향 "생태계 호환성 암묵 가정" 으로 확장 후보)
- pnpm 공식 문서: https://pnpm.io/cli/run — `pnpm run --if-present` 패턴 (pnpm 8+)
