---
title: Gemini cross-validate API capacity 소진 시 Claude 단독 폴백 + 후속 재시도 이슈 분리 패턴
type: report
source_repo: coseo12/harness-setting (v2.13.0 / v2.15.0 박제 직후 cross-validate 루틴)
source_issue: 40
captured_at: 2026-04-19
status: refined
tags: [cross-validate, gemini, capacity-exhaustion, claude-only-fallback, retrospective]
related:
  - ../meta/patterns/cross-validate-after-policy-freeze.md
  - ./cross-validate-accept-vs-defer.md
  - ./cross-validate-self-application-positive-path.md
  - ./fourth-automation-mapping-layer.md
---

## 배경/상황

harness-setting CLAUDE.md 에 "정책·설계·ADR·CRITICAL DIRECTIVE 박제 직후 1회 cross-validate 루틴" (volt #23) 이 박제되어 있음. v2.13.0 / v2.14.0 / v2.15.0 연속 릴리스 직후 Gemini API (gemini-3.1-pro-preview) 호출 시도했으나 **2회 중 2회 429 RESOURCE_EXHAUSTED 발생** ("No capacity available for model gemini-3.1-pro-preview on the server"). 첫 번째는 10회 retry 실패, 두 번째는 2회 retry 실패.

## 내용

#### 스킬 규칙 (기존)

CLAUDE.md 교차검증 섹션:
- "Gemini 실패 시 스킵하고 Claude 단독 분석 을 명시한다"
- "경량 모델 폴백은 하지 않는다 — 교차검증의 가치는 깊은 분석에 있다"

#### 실제 처리 (관찰)

1. **1차 실패 (v2.13.0 gitflow 복원 직후)**: retry 10회 실패 → Claude 단독 분석 실행. 3 개선안 도출 후 후속 이슈 (#104 #105 #106 #107) 로 박제
2. **1차 재시도 (v2.14.0 직전)**: `gemini -p "hello"` 성공 확인 후 본 검증 시도 → 또 429 retry 2회 실패 후 **응답 수신 성공** (부분 복구 상태였던 듯). Gemini 고유 발견 반영하여 #110 생성
3. **2차 실패 (v2.15.0 박제 직후)**: `gemini -p "hello"` 즉시 429 → Claude 단독 간략 분석으로 폴백. 추가 MUST 변경 없음 판정

#### Claude 단독 폴백의 한계

- 단일 모델 편향 노출 효율이 박제 직후에 가장 높다는 규칙 전제 (volt #23) 를 완전히 이행 못 함
- Claude 의 자기 비판은 "있는 것을 비판" 은 가능하나 "빠진 관점" 검출은 약함
- Gemini 는 이전 대화 맥락 없이 cold read 해서 전제 숨어있는 것을 찾아내는 경우 가 있음

## 교훈 / 다음에 적용할 점

1. **API capacity 소진 재시도 전략** — 첫 실패 후 즉시 Claude 단독 하지 말고 **최소 1회 지연 재시도** (예: `gemini -p "hello"` 로 빠른 체크 후 본 검증 시도). 2차 실패 시 Claude 단독 + 후속 이슈 박제
2. **Gemini 고유 발견 반영 3단 프로토콜 (volt #29) 과 결합** — 응답 성공 시 즉시 수용 vs 후속 분리 판단
3. **누락 가능성 있는 cross-validate 는 reminder 이슈로 박제** — harness #107 처럼 "API 복구 후 재시도" 이슈 생성. 후속에서 API 상태 확인 + 재시도 또는 close 판단
4. **박제 직후 루틴의 필수성은 API 의존성 없이 유지** — 단독 분석이라도 "지금 했음" 을 기록해야 "안 했음" 으로 오인되지 않음
5. **Gemini 2.5 Pro Preview (gemini-3.1-pro-preview) 는 free tier capacity 제한 엄격** — 연속 호출 시 빈번히 429. 중요 검증 전 capacity 체크 + retry 간격 (최소 30분) 권장

## 관련 노트/링크

- volt #23 (교차검증 루틴)
- volt #29 (고유 발견 수용 vs 분리)
- harness #107 (복구 후 재시도 이슈 — close 됨, 2차 성공)
- harness 릴리스 v2.13.0, v2.14.0, v2.15.0
