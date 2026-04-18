---
title: 정책/설계 박제 직후 교차검증 1회 루틴 — 단일 모델 편향 상쇄 패턴
type: pattern
source_repo: coseo12/harness-setting
source_issue: 23
captured_at: 2026-04-17
status: refined
tags: [cross-validate, gemini, semver, bias, meta-review, template-as-prompt]
related:
  - ../decisions/semver-behavior-vs-docs-policy.md
  - ../../notes/state-atomicity-multi-layer-defense.md
  - ../../notes/cross-validate-accept-vs-defer.md
  - ../../notes/gemini-capacity-fallback-retry-protocol.md
---

## 배경/상황

harness-setting v2.6.2 에서 SemVer 분류 정책을 확정·셀프 적용했다. Claude 단독 설계 시점에는 "문서/규약 추가 = PATCH" 로 일괄 분류하고 마쳤으나, 정책 박제 직후 Gemini 교차검증(cross-validate 스킬) 1회를 루틴으로 수행한 결과, 단일 모델이 놓친 관점을 발견해 정책을 세분화하게 됐다(v2.6.3).

## 내용

**Claude 단독 시점 정책 (v2.6.2)**

- PATCH = "코드 동작 변화 없는 모든 변경" (문서/규약/교훈/스킬 체크리스트 포함 일괄 PATCH)

**Gemini 교차검증에서 제기된 핵심 반론**

- 이 레포는 CLI + **프로젝트 템플릿(.claude/agents, .claude/skills, CLAUDE.md)** 혼합 배포이고, 다운스트림은 `harness update` 로 frozen 파일을 당겨 **에이전트가 프롬프트로 실행**한다.
- 즉 템플릿 파일은 일반 문서가 아니라 "AI 가 실행하는 프롬프트 소스코드". 에이전트 지시어·체크리스트 변경은 에이전트 **행동**을 바꾼다.
- "문서 = PATCH" 일괄 처리는 PATCH 자동 업데이트(`^2.6.0`)의 "안전한 호환성" 신뢰 모델을 위반한다.

**반영 결과 (v2.6.3)**

- PATCH/MINOR 경계 재정의: 에이전트 지시어·스킬 절차·체크리스트의 **행동 변화 = MINOR**, 행동 변화 없는 문서/문구/오타 = PATCH
- 단일 판정 질문 도입: **"이 변경으로 에이전트가 같은 입력에 다르게 동작하는가?"**
- CHANGELOG `### Behavior Changes` 섹션 의무화 (MINOR/MAJOR 필수, PATCH 도 frozen 파일 변경 시 `None — 문서/문구만` 명시)

**관찰된 패턴**

1. Claude 가 자기 산출물을 "완성" 으로 판정한 직후에도 외부 모델은 범주 오류(여기서는 "템플릿 = 일반 문서" 가정)를 찾아낸다.
2. 교차검증은 **설계 개선 루틴**으로 효과가 크다 — 토론 산출물 1회면 단일 모델 편향을 상당 부분 상쇄.
3. 반대로 Gemini 의 대안 중 일부(CalVer 전환, 모든 체크리스트 변경 = MINOR)는 과대 대응으로 판정되어 반려됨 — 교차검증은 **맹목 수용 금지**, Claude 재분석이 필수.
4. 교차검증 비용(Gemini 1회 호출)은 정책 변경 후 회귀 비용(다운스트림 에이전트 파이프라인 붕괴)에 비해 현저히 낮다.

## 교훈 / 다음에 적용할 점

- **정책·규약·설계 문서를 박제한 직후 교차검증 1회를 루틴화** — 특히 AI 에이전트/프롬프트 자산을 다루는 프로젝트에서 고효율. cross-validate 스킬이 이미 있다면 비용은 호출 1회 수준.
- **판정 체크리스트에 "외부 모델 관점 1회" 단계 삽입** — CLAUDE.md 또는 architect 에이전트 워크플로에 "박제 전 cross-validate 1회" 를 추가하면 편향 노출 빈도가 높아진다.
- **교차검증 결과는 Claude 가 재분석** — Gemini 산출도 모두 옳지 않다. 합의/이견/고유 발견을 분류하고, 과대 대응은 근거와 함께 반려.
- **"템플릿 = 실행 프롬프트" 프레임**은 AI 워크플로 템플릿을 배포하는 모든 프로젝트에 재사용 가능 — 버전 정책, 릴리스 노트, 자동 업데이트 전략을 이 관점에서 재검토할 가치.
- 본 패턴 적용의 실측 링크:
  - v2.6.2 정책 박제: [harness-setting#82](https://github.com/coseo12/harness-setting/pull/82)
  - 교차검증 → v2.6.3 세분화: [harness-setting#83](https://github.com/coseo12/harness-setting/pull/83), [#84](https://github.com/coseo12/harness-setting/pull/84)
