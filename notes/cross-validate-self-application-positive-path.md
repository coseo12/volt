---
title: 박제 직후 cross-validate 셀프 적용의 정상 경로 반복 성공 — 4회 연속 관찰
type: report
source_repo: coseo12/harness-setting (v2.16.0 / v2.17.0 / v2.18.0 / v2.19.0 연속 박제 직후 cross-validate 루틴)
source_issue: 42
captured_at: 2026-04-19
status: refined
tags: [cross-validate, gemini, dogfooding, self-application, fallback-protocol, positive-path-validation]
related:
  - ../meta/patterns/cross-validate-after-policy-freeze.md
  - ./gemini-capacity-fallback-retry-protocol.md
  - ./cross-validate-principle-declaration-bias.md
---

## 배경/상황

harness-setting CLAUDE.md 에 박제된 "정책·설계·ADR 박제 직후 1회 cross-validate 루틴" (volt [#23](https://github.com/coseo12/volt/issues/23)) + API capacity 폴백 프로토콜 (volt [#40](https://github.com/coseo12/volt/issues/40)) 이 **2026-04-19 세션 동안 4회 연속 실행** 되고 모두 정상 경로 성공 (429 미발동).

## 내용

#### 관찰 사례 4회

| PR | 박제 내용 | Gemini 응답 | 폴백 | reminder 이슈 |
|---|---|---|---|---|
| #113 (v2.15.1) | 3계층 방어 패턴 문서 (PATCH) | 정상 수신 | 미발동 | 조건 소멸 |
| #117 (v2.16.0) | auto-close 가드 + Gemini 429 폴백 선언 (MINOR) | 정상 수신 | 미발동 | 조건 소멸 |
| #126 (v2.17.0) | sub-agent 공통 JSON 스키마 Phase 1 (MINOR) | 정상 수신 | 미발동 | 조건 소멸 |
| #130 (v2.18.0) | cross_validate.sh 폴백 하드코딩 Phase 2 (MINOR) | 정상 수신 | 미발동 | 조건 소멸 |
| #134 (v2.19.0) | outcome JSON + architect 자동 매핑 Phase 3 (MINOR) | 정상 수신 | 미발동 | 조건 소멸 |

**5회 모두** capacity 체크 불필요 + claude-only fallback 미발동 + reminder 이슈 발동 조건 소멸.

#### 정상 경로 성공의 의미

단순히 "오류 없음" 이 아니라 **프로토콜의 정상 분기 자체가 검증** 됨:
- 정상 응답 수신 → `exit 0` 경로 실행 확인
- outcome JSON (v2.19.0 부터) 이 `"applied"` 기록
- reminder 이슈 발동 조건이 활성(앵커 설정) + 실패 조건 (폴백) 이 AND 로 묶여있음을 실증

폴백 경로는 스모크 테스트의 mock gemini 바이너리로 실증. 두 실증이 **상호 보완** — 실 API 정상 + mock 폴백 = 전 분기 커버.

#### 셀프 적용의 메타 특성

특히 PR #130 (v2.18.0 — 폴백 프로토콜 하드코딩 직후) 과 PR #134 (v2.19.0 — outcome JSON 하드코딩 직후) 의 cross-validate 호출이 **방금 박제된 규약을 자기 자신에게 적용** 함:
- v2.18.0 cross-validate: `CROSS_VALIDATE_ANCHOR=MINOR-behavior-change` + `GH_PR_CONTEXT=PR#130` 설정 → 방금 박제된 환경변수 규약 첫 사용
- v2.19.0 cross-validate: outcome JSON 을 생성하는 코드를 담고 있으므로 호출 자체가 outcome JSON 생성 시험

Gemini 가 v2.18.0 리뷰에서 이를 "dogfooding 전략 매우 우수" 로 평가.

## 교훈 / 다음에 적용할 점

1. **정상 경로 반복 성공은 조용한 실증** — 실패가 없다고 테스트 부족이 아니다. 정상 분기 자체가 프로토콜 행동을 증명
2. **셀프 적용은 박제 직후 1회로 충분** — 매 릴리스마다 셀프 적용을 강요하면 Gemini quota 낭비 + 피로 누적. 1회 성공 후 다음 박제까지 대기
3. **실 API 정상 + mock 폴백 = 전 분기 커버** — 실 API 로는 429 시나리오를 재현하기 어려우므로 mock 으로 폴백 경로 별도 검증. 두 실증이 **상호 보완** 관계
4. **"아무 일 없음" 기록 의무** — cross-validate 수행했고 정상 수신했다는 사실 자체를 PR 코멘트 + CHANGELOG Notes 에 박제 (volt [#40](https://github.com/coseo12/volt/issues/40) 교훈 4 연장). 향후 "cross-validate 불이행" 의혹 방지

## 관련 노트/링크

- volt [#23](https://github.com/coseo12/volt/issues/23) — 박제 직후 1회 cross-validate 루틴 원본
- volt [#40](https://github.com/coseo12/volt/issues/40) — Gemini capacity 소진 폴백 프로토콜
- harness releases [v2.15.1](https://github.com/coseo12/harness-setting/releases/tag/v2.15.1) ~ [v2.19.0](https://github.com/coseo12/harness-setting/releases/tag/v2.19.0)
- harness PR [#130](https://github.com/coseo12/harness-setting/pull/130) / [#134](https://github.com/coseo12/harness-setting/pull/134) — 가장 셀프 적용 메타성이 강한 두 사례
