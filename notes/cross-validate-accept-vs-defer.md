---
title: 교차검증 제안의 즉시 수용 vs 후속 분리 판단 기준
type: knowledge
source_repo: coseo12/harness-setting
source_issue: 29
captured_at: 2026-04-18
status: refined
tags: [cross-validate, gemini, scope-management, sprint-contract, decision-protocol, follow-up-issue]
related:
  - ./state-atomicity-multi-layer-defense.md
  - ../meta/patterns/phased-release-split-pattern.md
  - ../meta/patterns/cross-validate-after-policy-freeze.md
  - ./gemini-capacity-fallback-retry-protocol.md
---

## 요약

Gemini 교차검증이 제시한 제안을 현재 PR 에 즉시 반영할지, 후속 이슈로 분리할지 판단하는 **3단계 프로토콜**. volt [#23](https://github.com/coseo12/volt/issues/23) 의 교차검증 루틴을 보완한다. 핵심: **스프린트 계약(비목표 포함)이 Gemini 제안보다 우선**. 근본 해결책이라도 현재 스프린트 범위를 벗어나면 후속 이슈로 분리.

## 본문

#### 기존 루틴 (volt [#23](https://github.com/coseo12/volt/issues/23))

- 정책·ADR·CRITICAL DIRECTIVE 박제 직후 교차검증 1회 호출
- Gemini 산출물을 합의 / 이견 / 고유발견 으로 분류
- 과대 대응은 근거와 함께 반려

#### 공백 — "고유발견"의 처리 기준 부재

`[#23](https://github.com/coseo12/volt/issues/23)` 은 **반려 기준** 은 제공하지만 **수용 기준** 이 모호했다: Gemini 가 제안한 "근본 해결책" 이 현재 PR 에 포함될지, 후속으로 분리될지의 판단이 ad-hoc 했다.

#### 3단계 판단 프로토콜

**1. 합의 항목 선별** — 내 설계와 일치하는 Gemini 지적은 **현재 PR 에 반영**. 이견은 근거 비교 후 취사.

**2. 고유 발견의 범위 체크** — Gemini 만의 제안이라면 현재 스프린트 계약(특히 **비목표**) 과 대조:
  - 스프린트 범위 내 → 현재 PR 에 반영
  - 스프린트 범위 밖 (비목표와 상충) → **후속 이슈로 분리**
  - 판단 기준: "이 변경이 현재 PR 의 `Behavior Changes` 에 추가 항목을 만드는가? 그 항목이 원 완료 기준과 직교하는가?"

**3. 분리 시 박제 규칙** — 후속 이슈를 즉시 생성하여 맥락 유실 방지:
  - 이슈 본문에 Gemini 제안의 설계 스케치 인용
  - 원 PR 과 링크 (`Builds on: #원PR`)
  - 우선순위 초안 명시 (high / medium / low)

#### 실제 적용 사례 (harness [#89](https://github.com/coseo12/harness-setting/issues/89) → [#92](https://github.com/coseo12/harness-setting/issues/92))

Gemini 가 `previousSha256` 매니페스트 스키마 확장을 **근본 해결책** 으로 제안했다. 판단 흐름:

1. 합의 검사 → Gemini 지적이 내 설계(post-apply 검증)의 **한계** 를 정확히 짚음. 합의 수준 높음
2. 범위 체크 → 내 PR 의 비목표에 "매니페스트 스키마 변경 없음" 명시됨. 스키마 확장은 비목표와 **직접 상충**
3. 분리 결정 → 후속 이슈 [#92](https://github.com/coseo12/harness-setting/issues/92) 생성 (스프린트 계약 박제, 설계 스케치 포함)

**결과**: 3 PR / 3 릴리스로 자연 분할되어 각 단계 위험 독립, 사용자가 v2.8.0 관찰 후 v2.9.0 설계 재조정 기회 확보.

#### 반면 예시 — 즉시 수용 케이스 (반대 사례)

harness [#89](https://github.com/coseo12/harness-setting/issues/89) 교차검증에서 Gemini 가 "merge type 스킵 / managed-block 외부 편집 오탐 방지 테스트 추가" 도 제안. 이 중:
- merge 테스트: 의존성 복잡도로 분리 (후속 이슈) ← 후보 4 참조
- managed-block 테스트: **현재 범위 내 + 비용 저렴** → 즉시 수용 (Phase 2 에 포함)

즉, **고유 발견 안에서도 비용/범위에 따라 취사**.

#### 금지

- 스프린트 계약이 명시한 비목표를 **Gemini 제안이 타당하다는 이유만으로 무시** — CRITICAL #6 침범
- 분리 판단 없이 현재 PR 로 수용하여 스코프 팽창
- 분리 후 이슈 생성을 미루어 맥락 유실

## 관련 노트/링크

- volt [#23](https://github.com/coseo12/volt/issues/23) — 교차검증 루틴 원안 (이 노트는 해당 루틴의 수용/분리 기준 보완)
- harness [#89](https://github.com/coseo12/harness-setting/issues/89) 댓글 — 실제 교차검증 결과와 분리 결정
- harness [#92](https://github.com/coseo12/harness-setting/issues/92) — 분리된 후속 이슈의 실행 예시 (Phase 1 + Phase 2)
