---
title: 문서/규약 추가는 PATCH, 행동 변화는 MINOR — SemVer 판정 정책
type: report
report_type: decision
source_repo: coseo12/harness-setting
source_issue: 22
captured_at: 2026-04-17
status: inbox
tags: [semver, versioning, release, policy, decision]
related: []
---

## 배경/상황

harness-setting v2.5.0~v2.6.0 사이 볼트 반영(문서/규약 추가)을 MINOR 로 처리한 선례가 누적되면서, 실제 코드 동작 변화가 없는 변경만으로 마이너 버전이 빠르게 상승했다. v2.6.1 릴리즈에서 단일 규칙 추가를 PATCH 로 처리하며 기준이 불일치함을 인식 → 정책을 명시화하기로 결정.

## 내용 (v2.6.2 초안 정책)

**후보 비교**

| 방식 | 장점 | 단점 |
|---|---|---|
| 기존: 규약 추가 = MINOR | "의미 있는 변경"을 명시적으로 표시 | 패치성 변경 누적 시 버전 과상승, MINOR 가 기능 확장과 분리 |
| 신규: 규약 추가 = PATCH | MINOR 가 진짜 기능 확장만을 가리킴, SemVer 원칙 부합 | "문서 변경도 중요하다"는 신호가 약해짐 (CHANGELOG 로 보완) |

**결정 (v2.6.2)**

- **MAJOR**: 하위 호환을 깨는 변경 (CLI 인자 제거/시그니처 변경, 스킬·에이전트 계약 파괴, 설정 스키마 breaking)
- **MINOR**: 코드 동작이 포함된 신규 기능 추가 (신규 CLI 서브커맨드, 신규 에이전트/스킬, 신규 hook/automation, 신규 옵션)
- **PATCH**: **코드 동작 변화 없는 모든 변경** (문서/규약/규칙/CLAUDE.md 교훈 추가·수정, 기존 스킬 체크리스트 보강, 버그 수정, 문구 개선)
- 볼트 반영(문서/규약 추가)은 기본 PATCH. 누적이 크고 한 주제로 묶일 때만 MINOR 로 승격

**셀프 적용**

정책 변경 자체가 문서/규약 변경이므로 PATCH(v2.6.2) 로 릴리즈. 새 기준의 첫 적용 사례로 기록.

## 세분화 (v2.6.3 — Gemini 교차검증 반영)

v2.6.2 정책은 Gemini 교차검증에서 **조건부 통과** 로 판정됨 → v2.6.3 에서 세분화.

**추가된 보완점**

1. **행동 변화 vs 문서 변경 분기** — 에이전트 지시어·스킬 절차·체크리스트의 **행동 변화는 MINOR** 로 승격. PATCH 는 행동 변화 없는 문서/문구/오타/버그 수정으로 한정.
2. **판정 질문 단일화** — "이 변경으로 에이전트가 같은 입력에 다르게 동작하는가?" 예 = MINOR, 아니오 = PATCH.
3. **CHANGELOG `### Behavior Changes` 섹션 의무화** — MINOR/MAJOR 는 필수. PATCH 도 frozen 파일(`.claude/`) 변경 시 `None — 문서/문구만` 명시하여 자동 업데이트 신뢰 모델을 보호.
4. **소급 공지** — README/CHANGELOG 상단 NOTICE 블록으로 v2.6.2 정책 전환 가시화.

**Gemini 핵심 피드백 (합의된 부분)**

- 템플릿은 "AI 가 실행하는 프롬프트 소스코드" — 일반 문서와 동일 취급은 범주 오류.
- `harness update` 로 frozen 파일이 갱신되면 다운스트림 에이전트 **행동**이 바뀐다. PATCH 자동 업데이트 신뢰 모델 위반 리스크 실재.

**반려된 대안 (별도 ADR 로 미룸)**

- CalVer 전환 + 템플릿 패키지 분리 라이프사이클 — 현 배포 구조 대비 비용 과다. 추후 별도 의사결정.
- "모든 체크리스트/규약 변경 = MINOR" 차선책 — 오타·문서 변경까지 MINOR 로 올리면 원래 정책 폐기 동기(버전 과상승)와 모순.

## 박제 위치

- harness-setting CLAUDE.md `### 릴리스` 섹션에 구체 예시까지 명시
- 관련 PR: [#81](https://github.com/coseo12/harness-setting/pull/81), [#82](https://github.com/coseo12/harness-setting/pull/82), [#83](https://github.com/coseo12/harness-setting/pull/83) (세분화), [#84](https://github.com/coseo12/harness-setting/pull/84) (세분화 릴리스)
- 릴리즈: [v2.6.2](https://github.com/coseo12/harness-setting/releases/tag/v2.6.2), [v2.6.3](https://github.com/coseo12/harness-setting/releases/tag/v2.6.3)

## 교훈 / 다음에 적용할 점

- **애매 시 낮은 쪽 선택** — SemVer 판정이 애매할 때 기본값을 낮은 레벨(PATCH)로 두면 과상승을 예방하고, 대신 CHANGELOG 로 의미를 전달한다.
- **선례도 폐기 가능** — 과거 릴리즈 관행이 일관되지 않거나 과상승을 유발하면 명시적으로 폐기한 뒤 정책을 문서로 박제한다.
- **판정 기준은 단일 질문으로 좁혀야 운용 가능** — "에이전트가 같은 입력에 다르게 동작하는가?" 처럼 이분법을 한 문장으로 박제.
- **템플릿/지시어 = 코드** — AI 를 구동하는 프롬프트 텍스트는 일반 문서와 다르게 취급. 행동 변화 수반 시 MINOR 로 승격.
- **교차검증이 정책 세분화를 낳는다** — 자체 결정(v2.6.2) 후 Gemini 검증으로 보완점(v2.6.3)을 얻는 흐름이 유효했음. 중요 정책은 1차 결정 → 외부 검증 → 세분화 릴리즈 패턴을 기본 루틴으로 고려.
- 다른 프로젝트(에이전트/스킬 기반 CLI)에서도 동일 기준 적용 가능 — "행동 변화 = MINOR, 문서만 = PATCH" 이분법이 가장 읽기 쉬움.
