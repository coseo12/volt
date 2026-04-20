---
title: 다운스트림 실측이 최종 가드 — upstream 3중 방어 blindspot + 역방향 피드백 루프 설계
type: knowledge
source_repo: coseo12/harness-setting
source_issue: 60
captured_at: 2026-04-20
status: inbox
tags: [downstream-validation, final-guard, ci-coverage-limit, real-usage-as-test, production-as-canary, verification-asymmetry]
related:
  - ../meta/patterns/dogfooding-guard-self-violation-pattern.md
  - ../notes/ci-green-no-test-execution-trap.md
  - ../notes/external-tool-claim-empirical-verify.md
  - ../notes/release-version-metadata-drift-guard.md
---

## 요약

upstream 프로젝트의 **단위 테스트 / reviewer / Gemini cross-validate** 가 모두 통과해도, 다운스트림 **실 사용 환경** 에서만 드러나는 결함이 있다. harness-setting v2.28.2 / v2.29.0 pnpm `--if-present` 버그 (volt [#59](https://github.com/coseo12/volt/issues/59)) 가 3중 방어를 모두 통과했으나 astro-simulator#270 이 v2.28.2 적용 시점에 CI red 로 최초 발견한 사례. 원리 일반화: **다운스트림 실측은 upstream 테스트 계층이 구조적으로 커버할 수 없는 blindspot 의 최종 감지 지점**. 이를 적극 인정하고 **역방향 피드백 경로** 를 설계해야 upstream 품질이 다운스트림 운영으로 자가 회복한다.

## 본문

#### upstream 3중 방어가 모두 통과한 상태

harness-setting 의 pnpm `--if-present` 버그 (v2.28.2 도입 → v2.29.0 유지):

| 방어 계층 | 대상 | 결과 |
|---|---|---|
| 1. 자동 테스트 | `npm test` (56 pass) | ✅ 통과 — 본 저장소는 npm 으로 테스트해서 pnpm 경로가 실행조차 안 됨 |
| 2. 내부 reviewer | 정적 코드 리뷰 | ✅ 통과 — 패턴만 보고 실 동작 실측 불가 |
| 3. Gemini cross-validate | 외부 툴 주장 실측 가드 (volt #51) | ✅ 통과 — Gemini 도 "침묵" 상태, 주장하지 않으면 가드 미발동 |

3중 방어 모두 통과 상태에서 머지·릴리스. **다운스트림 astro-simulator#270** 이 pnpm 프로젝트에서 v2.28.2 적용 → `detect-and-test` CI red → 19분 내 보고 → v2.29.1 복구.

#### 왜 upstream 계층은 본 결함을 구조적으로 놓쳤는가

1. **"자기 저장소" 의 테스트 환경 한계** — harness-setting 자체는 npm 테스트만 돌림 (자기 저장소 `package.json` 에 lockfile 없음). pnpm/yarn/poetry/pipenv 경로는 **자기 CI 에서 실행조차 안 됨** — 구현했으나 실측 불가
2. **reviewer / cross-validate 는 코드 정합성 판단** — 런타임 동작은 환경 의존. 같은 CI 를 pnpm 프로젝트에서 돌렸을 때 flag forwarding 이 어떻게 되는지는 Node.js / pnpm / 각 test runner 의 상호작용 결과
3. **upstream 프로젝트의 CI 매트릭스 한계** — 모든 다운스트림 시나리오 (pnpm 모노레포 × vitest / jest / mocha × workspaces × 각종 script 조합) 를 upstream 에서 매트릭스로 돌리면 CI 비용 수십 배 증가

결론: upstream 에서 이 결함을 **원리적으로 감지할 수 없는 blindspot** 이 존재. 다운스트림 실측이 "최후의 가드" 가 아니라 **유일하게 가능한 가드** 인 경우가 있다.

#### 역방향 피드백 경로의 중요성

upstream 이 결함을 못 잡는다고 release 자체를 막으면 개발이 멈춘다. **실용적 해결책**: release 를 막지 않되, 다운스트림 실측 결과가 **빠르게 upstream 으로 역류** 하는 경로 설계:

| 요소 | 본 사례 성공 요인 | 일반화 규칙 |
|---|---|---|
| **신속 감지** | astro-simulator CI 가 v2.28.2 적용 직후 red | 다운스트림은 upstream 적용 즉시 CI 실행 — 지연 최소화 |
| **명확 보고** | harness#181 이슈에 증상·원인·제안 모두 포함 | 다운스트림이 "사용자" 가 아니라 "공동 QA" — 원인까지 제시 |
| **짧은 fix 주기** | #182 fix → #183 release → v2.29.1 총 ~6분 | upstream 이 긴급 PATCH 경로 사전 준비 (CHANGELOG 템플릿, 릴리스 가드 스크립트 등) |
| **회귀 테스트 추가 여지** | (미실행) 본 버그를 재현하는 fixture 를 upstream 통합 테스트에 추가 가능 | 다운스트림 실측 결과를 upstream 의 영구 가드로 승격 — 같은 결함 재발 방지 |

#### 일반화된 패턴 — "다운스트림 → upstream 피드백 루프"

본 패턴이 작동하려면 3가지 전제 필요:

1. **downstream 이 upstream 을 자동/빠르게 업데이트** (본 harness 는 `harness update` CLI + 매니페스트 시스템 — 적용이 low friction)
2. **downstream 이 자체 CI 가 있음** (astro-simulator 가 자기 CI 를 운영 중 — 업데이트 후 자동 감지 가능)
3. **downstream → upstream 보고 경로가 열려 있음** (GitHub issue / 같은 운영자 / Slack 등)

전제가 깨지면 다운스트림 실측이 **감지되더라도 upstream 에 전달되지 않음** → upstream 은 결함을 모른 채 계속 릴리스.

#### 다른 적용 시나리오

- **Rust crate 릴리스 후 downstream Cargo 빌드 실패** — crate 자체 테스트는 pass, 특정 조합의 feature flag 에서 depedency resolution 문제
- **npm 패키지 릴리스 후 downstream React 앱 런타임 에러** — 라이브러리 자체 테스트는 pass, 특정 React 버전 / SSR 환경에서 문제
- **Docker 이미지 태그 업데이트 후 downstream Kubernetes Pod crash** — 이미지 자체 smoke test pass, 특정 노드/리소스 제약에서 재현
- **DB schema migration 릴리스 후 downstream ETL 파이프라인 실패** — migration 자체 test pass, 특정 규모의 실 데이터에서 timeout
- **LLM 모델 업데이트 후 downstream agent 행동 변화** — 벤치 통과, 특정 사용자 워크플로에서 regression

공통 조건 (본 패턴 적용 가능 신호):
- upstream 의 자기 테스트 환경이 downstream 의 **사용 환경 매트릭스 전체를 커버할 수 없음** (구조적 비대칭)
- downstream 이 자동 CI 또는 관찰 인프라 보유
- 양방향 보고 채널 존재

#### upstream 이 할 수 있는 사전 방어 3가지

본 패턴이 불가피하다면 upstream 이 **다운스트림 실측 피드백 속도** 를 최대화해야:

1. **긴급 PATCH 파이프라인 템플릿** — CHANGELOG 엔트리 템플릿 / hotfix 브랜치 규약 / 태그 자동화. 본 harness 의 `Release vX.Y.Z` PR 형식 표준화가 좋은 예
2. **다운스트림 명시 — 대표 다운스트림 / 업데이트 경로 문서화**. 예: "이 버전을 먼저 적용하는 다운스트림: astro-simulator / portfolio-26". 보고 루트 명시
3. **회귀 가드 소급 승격** — 다운스트림에서 감지된 결함을 재현하는 테스트 fixture 를 upstream 에 통합. 본 harness 의 "암묵 관례의 구조적 승격" 패턴 (volt [#56](https://github.com/coseo12/volt/issues/56)) 과 동일

## 교훈 / 다음에 적용할 점

1. **"자기 저장소 테스트가 통과" 는 다운스트림 사용에서의 정상 동작 증명 아님** — volt #48 "CI 통과 ≠ 테스트 실행" 의 하드 한계 형태. 심지어 실 테스트가 돌아도 환경 매트릭스 차이로 다운스트림 결함 존재 가능
2. **다운스트림을 "사용자" 가 아닌 "공동 QA" 로 대우** — 보고 품질을 높이는 신호 (이슈 템플릿, 재현 단계 명시, 제안 수정안 포함) 를 upstream 이 요구하고 지원
3. **다운스트림 실측은 모든 upstream 가드의 hierarchical 후행** — upstream 의 모든 테스트·리뷰·cross-validate 가 통과해도 다운스트림 실측이 최종 검증. 따라서 upstream 은 "감지 불가능한 blindspot" 을 인정하고 **역방향 피드백 속도** 를 최적화
4. **피드백 루프 인프라는 upstream 품질의 일부** — 다운스트림 프로젝트를 upstream 개발자가 관리하지 않는 경우에도 보고 채널 / 이슈 템플릿 / 응답 속도가 upstream 의 장기 품질을 결정

## 관련 노트/링크

- 본 사례 원인 이슈: volt [#59](https://github.com/coseo12/volt/issues/59) (pnpm `--if-present` 가드 셀프 위반)
- 다운스트림 최초 감지: astro-simulator#270 (v2.28.2 적용 시 CI red)
- upstream 수정 체인: [harness#181](https://github.com/coseo12/harness-setting/issues/181) → [#182](https://github.com/coseo12/harness-setting/pull/182) → [#183](https://github.com/coseo12/harness-setting/pull/183) (v2.29.1)
- 선행 volt [#48](https://github.com/coseo12/volt/issues/48) (CI 통과 ≠ 테스트 실행), [#51](https://github.com/coseo12/volt/issues/51) (외부 툴 주장 실측), [#56](https://github.com/coseo12/volt/issues/56) (암묵 관례 구조적 승격)
