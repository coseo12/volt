---
title: 측정법 검증 우선 원칙 4단계 확장 — 데이터 신뢰성 재확인 (#32 연장선)
type: pattern
source_repo: coseo12/astro-simulator
source_issue: 53
captured_at: 2026-04-20
status: refined
tags: [measurement-methodology, sprint-contract, data-integrity, debug-workflow, volt-32-extension]
related:
  - ../../notes/dod-fail-measurement-verify-first.md
---

## 배경/상황

[volt #32](https://github.com/coseo12/volt/issues/32) 에서 박제한 "수치 DoD 미달 시 측정 방법 검증 우선" 원칙의 기존 3단계는 **식 수정 → 적분 조건 검증 → 알고리즘 교체** 순이다. astro-simulator P9 PR-2 (#260) 에서 D5-b (Laplace 위상 진폭 ±2°) 미달 진단 중, **3단계 전수 수행 후에도 미달이 지속되는 새 패턴** 이 관찰됨.

- **관찰 사례**: `measure_laplace_resonance()` 가 관찰 기간 내 peak-to-peak **471°/1200°** 반환 (평형 근방 진동 아닌 circulation)
- **1단계 식 수정 2회 수행**: 근점 검출 → mean motion 선형 회귀 (D1~D4 통과 확인) + osculating λ=Ω+ω+M → true longitude atan2 (detrend 잔차 개선)
- **2단계 적분 조건 검증**: 50/100/200 Io 주기 비교, Yoshida 4차 dt=60s stability 확인 — 기존 P7 ADR 근거
- **3단계 (알고리즘 교체 고려)**: peak-to-peak → RMS → FFT 전환 검토 — ADR §재검토 조건 #2 박제
- **결과**: 모든 단계 수행 후에도 미달. 측정 도구 자체는 **libration 재현 가능** 확인 (원궤도 fixture e=0 + synthetic 평형 데이터에서 peak-to-peak/2 ≤ 1° 관찰)

## 내용

#### 근본 원인 발견

PR-1 (#258) 에서 박제한 `solar-system.json` 의 Galilean 4체 `meanLongitudeDeg` 값으로 계산한 **초기 Laplace 인자 φ₀ = 218°**. 이론 평형점 **180°** 대비 **38° 벗어남** → libration 경계를 넘어 **circulation 영역**. JPL Horizons 원본 값의 epoch 일치 여부 또는 좌표계 정의 차이로 추정. 측정 도구·적분기·식 모두 정상.

#### 패턴 추상화: 측정법 검증 우선 원칙의 **미완 단계**

기존 3단계는 **"측정 도구 (코드 + 적분)"** 범위만 검증한다. 하지만 DoD 미달 원인은 다음과 같이 양분될 수 있다:

1. **도구 측**: 식·샘플링·적분기·알고리즘 — 기존 3단계 커버
2. **입력 측**: 초기 조건·fixture·외부 참조 데이터 — **기존 3단계 미커버**

도구 측 3단계를 전수 수행한 후에도 미달이면 **입력 측 검증** 이 4단계로 필요하다.

#### 4단계 제안: 데이터 신뢰성 재확인

**조건**: 기존 3단계 (식 / 적분 조건 / 알고리즘) 전수 수행 후에도 DoD 미달이 지속되고, **측정 도구가 이상 상태 (원궤도 / synthetic 데이터) 에서는 예상 동작 확인됨**.

**절차**:
1. **초기 조건 출처 재확인** — fixture (JSON / 상수 테이블 등) 의 **발행 주체·epoch·좌표계·단위** 원본 문서 대조
2. **이론 평형/경계값 계산** — 측정 대상의 이론 평형점 / 한계 임계값을 독립 계산하여 fixture 값이 **정상 영역 내** 에 있는지 검증
3. **발견 시 분리** — 데이터 이슈로 판정되면 후속 이슈로 분리 (현 스프린트 범위 밖) + 코드 assertion 제거 + `#[ignore]` 유지 + 세 위치 박제 (CLAUDE.md §스프린트 계약 7)

**의사결정 질문**:
- "측정 도구가 synthetic / 이상 fixture 에서 예상 동작하는가?" — 도구 정상 확인
- "fixture 값이 측정 대상의 이론 평형/경계 내에 있는가?" — 데이터 신뢰성 확인
- 둘 다 만족 안 하면 도구 측 재검토 (기존 3단계 재시작)

## 교훈 / 다음에 적용할 점

#### 다른 프로젝트 적용 패턴

1. **물리 시뮬레이터**: 관측량 측정 미달 시 fixture epoch / 좌표계 / 단위 재확인
2. **ML 모델 평가**: metric 미달 시 데이터셋 (train/val split, label noise, sampling bias) 재확인 후 모델 구조 수정
3. **성능 벤치마크**: 목표 미달 시 benchmark fixture (workload, dataset size, hardware) 와 실제 production 비교 — 도구 (측정 방식) vs 입력 (fixture) 양분
4. **API 계약 테스트**: 응답 불일치 시 mock/stub 데이터 vs 실제 endpoint 응답 비교

#### 프레임워크 규범 반영 권고

CLAUDE.md / agents/*.md §스프린트 계약 10 업데이트:

```
수치 DoD 미달 시 측정 방법 검증 우선:
(0) 측정 방법 검증 → (1) 식/구현 수정 → (2) 알고리즘 교체 → **(3) 데이터 신뢰성 재확인 (신설)**
(3) 은 (0)~(2) 전수 수행 + 측정 도구가 synthetic 데이터에서 정상 동작 확인 후에만.
데이터 이슈 발견 시 후속 이슈로 분리, 코드 assertion 제거 + #[ignore] + 세 위치 박제.
```

#### 세 위치 박제 규약 (CLAUDE.md §스프린트 계약 7) 의 실증 사례

astro-simulator P9 D5-b:
- 코드 주석: `laplace.rs:551` `#[ignore = "known-issue; JPL λ offset 으로 공명 평형 미성립"]`
- PR 본문: #260 Known Issue 섹션
- CHANGELOG: `[0.9.0]` §알려진 제한
- follow-up: [astro-simulator #261](https://github.com/coseo12/astro-simulator/issues/261)

## 참고

- 선행: [volt #32](https://github.com/coseo12/volt/issues/32) — 수치 DoD 미달 시 측정 방법 검증 우선 (3단계 원칙)
- astro-simulator #260 (P9 PR-2 Rust satellites, 관찰 원천)
- astro-simulator #261 (D5-b 데이터 교정 follow-up)
- ADR [`docs/decisions/20260420-p9-galilean-laplace-rings.md`](https://github.com/coseo12/astro-simulator/blob/main/docs/decisions/20260420-p9-galilean-laplace-rings.md) §재검토 조건 #2
