---
title: Claude 편향 5종 후보 — '생태계 호환성 암묵 가정' (1회 관찰, 3회 도달 대기)
type: knowledge
source_repo: coseo12/harness-setting
source_issue: 61
captured_at: 2026-04-20
status: inbox
tags: [claude-bias, checklist-extension, ecosystem-assumption, cli-flag-compat, bias-candidate, compile-rule-tracking, self-observation]
related:
  - ../notes/cross-validate-principle-declaration-bias.md
  - ../meta/patterns/dogfooding-guard-self-violation-pattern.md
---

## 요약

CLAUDE.md `## 교차검증` 에 박제된 **Claude 자체 편향 4종 체크리스트** (volt [#55](https://github.com/coseo12/volt/issues/55), v2.28.0) 에 **5번째 후보 편향: "생태계 호환성 암묵 가정"** 이 관찰됨. pnpm `--if-present` 가드 셀프 위반 (volt [#59](https://github.com/coseo12/volt/issues/59)) 에서 "npm 과 pnpm 은 같은 Node.js 생태계니 `--if-present` 도 동일 동작" 이라는 **CLI 호환성 암묵 전제** 가 드러남. 1회 관찰로 즉시 CLAUDE.md 박제는 compile 규약 ("3회 이상") 미충족 — 본 이슈는 **후보 관찰 1회** 박제 + 동일 패턴 2회 추가 관찰 시 규약 박제 승격 준비.

## 본문

#### 기존 Claude 편향 4종 (CLAUDE.md 현행)

| # | 편향 | 징후 | 보정 방향 |
|---|---|---|---|
| 1 | 낙관적 일정 산정 | "3일이면 충분" | 낙관 × 1.5~2 + 리서치 phase 분리 |
| 2 | 결합 관계 간과 | "A 와 B 병렬 가능" | 결합 감지 시 직렬 배치 |
| 3 | 폐기 프레이밍 선호 | "영구 폐기" | 재도입 트리거 명시 |
| 4 | 순수주의 원칙 적용 | 원칙 디폴트 강제 | 접근성 보장 형태 |

#### 후보 #5 — "생태계 호환성 암묵 가정" (신규 관찰 1회)

**징후**: 같은 언어/생태계에 속하는 도구들이 CLI flag / 설정 포맷 / 동작 모델을 **호환 공유** 한다고 실측 없이 가정.

**본 관찰 사례** (volt #59):
- npm `--if-present` 플래그를 pnpm 경로에 **실측 없이 복사**
- 실제: npm 은 네이티브 `--if-present` 지원 / pnpm 은 script args 로 forward (구조적 차이)
- 결과: 다운스트림 CI 실패 → v2.29.1 복구

**사전 감지 질문 (잠정)**:
- "이 PR 에서 **새로 도입** 하는 외부 도구 (CLI / 프레임워크 / 라이브러리) 가 있는가?"
- "각 도구의 flag / 설정 / 동작이 공식 문서에 명시되어 있는가?"
- "같은 생태계 (Node.js / Python / Rust 등) 라는 이유로 다른 도구의 flag 를 **복사** 하고 있는가?"

**보정 방향 (잠정)**:
- **같은 생태계 내 도구 간 flag 호환 가정 금지** — 각 도구 개별 검증
- **공식 문서 (CLI `--help` 출력 / 공식 README / changelog) 직접 인용** — LLM training data 캐시 의존 금지
- cross-validate 호출 시 `### 본 PR 이 도입한 외부 도구 flag 목록 + 각 공식 지원 여부` 명시 질문 삽입

#### 기존 편향과의 구분

본 후보는 편향 #1 (낙관적 일정) ~ #4 (순수주의) 중 어디에도 깔끔히 매핑되지 않음:

- 편향 #1 은 **시간 추정 축** — 본 후보는 **기술 호환성 축**
- 편향 #2 는 **작업 간 결합** — 본 후보는 **도구 간 기능 호환**
- 편향 #3 은 **결정의 시간적 유효성** — 본 후보는 **지식의 실시간 정확성**
- 편향 #4 는 **원칙 적용 강도** — 본 후보는 **암묵 전제 수용**

따라서 **독립 축** 으로 추가할 가치. 단 compile 규약 "3회 이상 관찰" 미충족이므로 본 이슈는 **후보 박제 + 추적** 만 수행.

#### 관찰 추적 목록 (본 이슈에 누적)

| 관찰 # | 날짜 | 저장소/PR | 사례 | 상태 |
|---|---|---|---|---|
| 1 | 2026-04-20 | harness#176 / #178 | npm `--if-present` → pnpm 실측 없이 복사 | ✅ 본 이슈 박제 |
| 2 | ? | — | (관찰 대기) | — |
| 3 | ? | — | (관찰 시 CLAUDE.md 편향 5종 확장 승격) | — |

**관찰 2 / 3 유사 후보 시나리오 (예측)**:
- `pip install -e .[dev]` 같은 pip extras 구문을 poetry 에 복사
- `tsconfig.json` 의 `compilerOptions` 를 `jsconfig.json` 에 그대로 복사 후 일부 옵션 미지원
- `Cargo.toml` 의 `[features]` 구문을 pyproject.toml 의 extras 에 유추 복사
- `.eslintrc` extends 를 `biome.json` 에 유추 복사
- GitHub Actions `uses:` vs CircleCI orb 의 버전 고정 문법 호환 가정

이런 관찰이 2회 더 발생하면 CLAUDE.md 편향 체크리스트 #5 로 승격. 본 이슈에 관찰 누적 기록 + 관련 PR 링크 코멘트 추가.

#### 1회 관찰의 박제 근거

compile 규약은 "3회 이상 관찰 또는 재발 시 손실 큰 종류만 행위 규칙으로 박제". 본 이슈는 **행위 규칙 박제가 아닌 후보 추적** 이므로 규약과 충돌하지 않음:

- **박제 형태**: knowledge 이슈로 관찰 기록만 (CLAUDE.md 수정 없음)
- **사용 목적**: 관찰 2/3 감지 시 본 이슈의 추적 목록에 누적 → 3회 도달 시 CLAUDE.md 승격
- **편향 #1~4 와 차이**: 편향 #1~4 는 volt #55 에서 **단일 세션 4회 동시 관찰** 로 규약 3회 충족. 본 후보는 **단일 관찰** 로 누적 대기

## 교훈 / 다음에 적용할 점

1. **편향 체크리스트 확장은 "관찰 누적 추적" 형태로 사전 관리** — 3회 도달까지 관찰을 흩어지지 않게 한 이슈에 누적. 본 이슈가 그 컨테이너
2. **"1회 관찰 즉시 박제" 는 가드 자기 위반에만 적용** (volt #59 규약 예외) — 후보 편향 관찰은 신중하게 3회 규약 따르기
3. **사전 감지 질문 시험 기회** — 본 이슈의 "사전 감지 질문 3개" 를 **cross-validate 호출 프롬프트에 실험적으로 삽입** 해보고, 3회 관찰 시점에 실효성 데이터 확보
4. **편향 축 분리 검증** — 새 후보가 기존 4축에 매핑되지 않는지 매번 확인. 매핑되면 기존 축 보강, 매핑 안 되면 신규 축 후보

## 관련 노트/링크

- 본 관찰 원천: volt [#59](https://github.com/coseo12/volt/issues/59) (pnpm `--if-present` 셀프 위반)
- 기존 편향 4종 박제: volt [#55](https://github.com/coseo12/volt/issues/55) (v2.28.0 에 CLAUDE.md 반영)
- compile 규약: harness [docs/knowledge-compilation.md](https://github.com/coseo12/harness-setting/blob/main/docs/knowledge-compilation.md)
- CLAUDE.md `## 교차검증` 의 Claude 편향 4종 체크리스트 (v2.28.0 이후)
