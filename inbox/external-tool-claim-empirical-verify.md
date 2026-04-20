---
title: cross-validate 오탐 2건 — 외부 툴 동작 주장은 실측 필수 (Gemini)
type: knowledge
source_repo: coseo12/harness-setting
source_issue: 51
captured_at: 2026-04-20
status: inbox
tags: [cross-validate, gemini, false-positive, external-tool, verification, blind-acceptance]
related:
  - ../notes/lint-staged-gitignore-silent-revert.md
  - ../meta/patterns/cross-validate-after-policy-freeze.md
---

## 요약

Gemini cross-validate 의 "개선 제안" 이 **외부 툴의 세부 동작** (예: `actions/setup-node@v4` 의 cache 동작) 에 관한 주장일 때는 **실측 없이 맹신 금지**. 이번 세션에서 2건 관찰: (A) setup-node `cache:'npm'` + lockfile 부재 자동 skip 주장 실측 반증, (B) body null guard 이미 존재하는데 "없으니 추가" 오탐. CLAUDE.md "맹목 수용 금지" 원칙의 **외부 툴 동작 카테고리 실증**.

## 본문

#### 관찰 사례 A — setup-node cache 동작 주장 반증

- 맥락: PR [#158](https://github.com/coseo12/harness-setting/pull/158) (v2.24.0, CI `npm test` 복구)
- Gemini 지적:
  > "setup-node@v4는 `cache: 'npm'`이 설정되어 있어도 `package-lock.json` 파일이 없으면 캐싱을 알아서 건너뛰도록 설계되어 있습니다. 따라서 조건을 분기할 필요가 없습니다."
- 원 코드: setup-node step 을 lockfile 유무로 2개 분기 (cache 활성 / 비활성)
- 수용 반영 → CI 실패:
  ```
  ##[error]Dependencies lock file is not found in /home/runner/work/harness-setting/harness-setting.
  Supported file patterns: package-lock.json,npm-shrinkwrap.json,yarn.lock
  ```
- 실측 결과: Gemini 주장과 반대로 **lockfile 부재 + `cache:'npm'`** 는 실제 `##[error]` 로 step 실패. warning 이 아닌 fail
- 대응: 단일화 커밋 revert + 원 2 step 분기 복원 + 실측 근거 주석 박제

#### 관찰 사례 B — 이미 존재하는 guard 를 "추가하라" 오탐

- 맥락: PR [#147](https://github.com/coseo12/harness-setting/pull/147) (v2.22.0, pr-review.yml 수정)
- Gemini 지적: "PR body 가 null 일 수 있으므로 `const body = context.payload.pull_request.body || '';` 추가 권장"
- 현행 코드:
  ```js
  const body = context.payload.pull_request.body || '';
  ```
- 즉 **제안 코드와 현행 코드가 완전 동일**. Gemini 가 diff 만 읽고 base 파일 전체를 확인하지 않아 guard 존재를 인지 못한 것
- 대응: 오탐 기각, PR 수정 없이 진행

#### 범용화 — 외부 툴 동작 주장에 대한 검증 루틴

| Gemini 주장 카테고리 | 검증 필수도 | 검증 방법 |
|---|---|---|
| 문법 / 논리 오류 | 중 | 로컬 run / unit test |
| 가독성 / 리팩토링 | 저 | 선택적 수용, 리뷰 |
| **외부 툴 동작 / CI / 프레임워크 기본값** | **최고** | **실측 (CI run, 샌드박스)** 필수 |
| 프로젝트 내부 구조 참조 | 중-고 | base 파일 전체 확인 (diff 만 보지 말고) |
| 보안 / 성능 | 고 | 테스트 + 프로덕션 유사 환경 |

#### 수용 전 체크리스트 (외부 툴 관련 제안)

1. **툴 공식 문서 확인** — Gemini 가 주장한 동작이 공식 문서에 명시되어 있는가? 추측성 기술 (e.g., "알아서 건너뛰도록 설계") 가드 필요
2. **CI/샌드박스 실측** — 반영 전 별도 커밋 / draft PR 로 동작 확인
3. **revert 가능한 단위로 커밋** — 실측 반증 시 롤백이 용이하게 작은 단위 커밋
4. **오탐 근거 박제** — revert 커밋 메시지 + 파일 주석 + CHANGELOG Notes 3곳에 이유 명시. 미래 기여자가 "왜 이 분기가 필요한가" 를 재발굴 시 빠르게 참조

#### diff-only 리뷰의 한계 (사례 B)

- LLM 코드 리뷰는 diff context 만 받는 경우 **base 파일의 기존 코드를 볼 수 없음**
- "X 가 없으니 추가하자" 류 제안은 diff 맥락에서 X 가 이미 base 에 있을 수 있음
- 대응: Gemini 제안을 수용하기 전 **관련 섹션의 base 파일 전체 grep** 으로 기존 구현 존재 여부 확인
- 프롬프트 개선 여지: cross-validate 스크립트가 PR diff 뿐 아니라 **관련 파일의 full content** 도 함께 전달하면 사례 B 같은 오탐 감소 가능

#### 인접 원칙

- CLAUDE.md `## 교차검증` "맹목 수용 금지", "결과는 Claude가 재분석" 의 실증
- CLAUDE.md "고유 발견의 수용 vs 후속 분리 3단 프로토콜" 중 1단계 (합의 선별) 를 통과하려면 **실측 검증** 이 포함되어야 함을 이번 세션이 보여줌
- volt [#13](https://github.com/coseo12/volt/issues/13) "staging 성공 ≠ 커밋 내용" 과 병렬 — AI/파이프라인 출력의 자신감과 실측의 간극
