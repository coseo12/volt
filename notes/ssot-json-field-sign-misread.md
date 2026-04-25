---
title: SSoT JSON 필드명 부호 규약 역해석 — 메인 오케스트레이터 자기 점검
type: knowledge
source_repo: coseo12/astro-simulator
source_issue: 73
merged_captures: [75]
captured_at: 2026-04-25
status: refined
tags: [ssot-json, sign-convention, main-orchestrator, self-check, report-first, bench-metric, bench-regression, sub-agent-reporting]
related:
  - ./dod-fail-measurement-verify-first.md
  - ./sub-agent-runtime-ssot-variance.md
  - ./pm-multiturn-dod-id-content-swap.md
---

## 요약

Developer sub-agent 가 반환한 SSoT JSON 의 `D4_regression_pct` 필드값 (예: `"idle": 366.2`) 을 메인 오케스트레이터가 **"+366% 회귀"** 로 역해석. 실제는 **+366% 개선** (fps 21.23→98.97). 리포트 본문은 `+366.2% 개선` 으로 명확히 표기되어 있었으나 JSON 필드명 `regression_pct` 단어에 끌려 본문 확인 없이 부호 규약 역해석. **SSoT JSON 필드명은 부호 의미를 역전시킬 수 있다** — 메인은 필드명만으론 부호 규약 판정 불가, 리포트 본문 선행 검증 필수. CLAUDE.md "수치 DoD 미달 시 측정 방법 검증 우선" 의 **메인 오케스트레이터 버전** — 판정 자체가 틀렸을 가능성 선행 의심.

> 본 노트는 동일 사건에 대한 두 차례 캡처(volt#73, volt#75)를 통합한 결과다. 첫 캡처(#73)는 메인 오판 원인 분석·예방 체크리스트에 비중, 보강 캡처(#75)는 SSoT JSON 필드 설계 규약(`*_pct` 절대값 + 부호 별도 필드) 제안을 추가했다. 두 캡처의 차이는 본문 안에 모두 반영했다.

## 본문

#### 현상

astro-simulator P11-B.2 PR #322 에서 developer sub-agent 가 SSoT JSON 반환:

```json
{
  "extends": {
    "D4_regression_pct": {
      "idle": 366.2,
      "play-1d": 342.1,
      "play-1y": 354.1,
      "focus-earth": -0.86,
      "focus-neptune": -3.93
    }
  }
}
```

메인 오케스트레이터가 이 값을 **"회귀율"** 로 역해석 → "idle/play-1d/play-1y 에서 +366% 회귀 발생 → DoD '회귀율 < 5%' 의 엄청난 위반" 으로 판정 → DoD 위반 경고 → PR draft 전환 + 측정법 재검증 지시 직전까지 진행.

사용자가 "D4 측정법 재검증" 선택지를 선택한 후 메인이 bench 리포트 본문 (`docs/benchmarks/p11-b-bench-20260424.md`) 을 **처음으로** 읽음:

```markdown
| idle | 21.23 | 98.97 | +77.74 | **+366.2% 개선** | ✓ |
```

부호 규약 실제: **양수 = fps 증가 = 개선 / 음수 = fps 감소 = 회귀**. DoD "회귀율 < 5%" 는 음수 값에만 적용. 5/5 scenario 모두 PASS 였고 최대 회귀는 focus-neptune -3.93%. Developer 보고 "5 scenario 전체 PASS, max 회귀 -3.93%" 가 정확했고 메인 판정이 틀렸음.

#### 메인 오판 원인

- SSoT JSON 필드명 `regression_pct` 단어가 "양수 = 회귀" 암시
- 리포트 본문 확인을 **생략** 하고 JSON 값만으로 판정
- sub-agent 보고의 "5/5 scenario 전체 PASS, max 회귀 -3.93%" 요약도 **있었으나** 메인이 JSON 값과 모순으로 간주하고 JSON 을 ground truth 로 채택

#### 일반화된 교훈

1. **SSoT JSON 필드명이 부호 의미를 역전시킬 수 있다**
   - `regression` 필드인데 양수가 개선 의미 — 필드명만으론 부호 규약 판정 불가
   - 필드명이 `change_pct` 나 `delta_pct` 처럼 중립적이었으면 덜 오도될 수 있었음
   - 규약: SSoT JSON 필드명이 의미 단어 (regression, error, loss 등) 를 포함할 때는 **반드시** 리포트 본문의 부호 규약 확인

2. **수치 판정 전 리포트 본문 선행 확인 루틴**
   - SSoT JSON 은 빠른 요약, 리포트 본문은 ground truth
   - 수치 DoD 판정 시 본문 읽기가 **선행 단계** 로 고정되어야 함
   - 특히 부호 중요한 수치 (회귀율, diff %, fps 변화) 는 본문 선행
   - "측정 방법 검증 우선" (CLAUDE.md 스프린트 계약 #10) 의 **메인 오케스트레이터 버전**

3. **메인 오케스트레이터도 자기 과대/과소 평가 대상**
   - "AI가 자기 작업을 과도하게 평가" 원칙은 sub-agent 뿐 아니라 메인에게도 적용
   - 메인이 "나는 전체 조망하니 판단이 정확하다" 편향
   - 수치가 예상 범위 밖일 때 (366% 같은 극단값) **측정법 / 해석법 먼저 의심** — 결과를 곧바로 DoD 위반으로 확정 금지

4. **sub-agent 보고와 JSON 의 모순은 탐색 신호**
   - sub-agent 가 "5/5 PASS" 라 보고했는데 JSON 값이 DoD 위반처럼 보이면, **둘 중 하나가 틀렸다** 는 신호 (그리고 흔히 해석이 틀린 경우가 많음)
   - 이런 모순은 본문 확인의 트리거

#### 예방 루틴 (메인 오케스트레이터용)

##### 운영 체크리스트

- [ ] SSoT JSON 수치 해석 전 리포트 본문 **존재 여부 + 경로** 확인
- [ ] SSoT JSON 필드 읽기 전 리포트 본문 linked path 먼저 cat/read
- [ ] 수치가 극단값 (>100% 또는 <-50% 등) 이면 부호 규약 재확인
- [ ] sub-agent 텍스트 요약과 JSON 값이 모순이면 본문 읽기 **선행**
- [ ] 부호 중요 수치 (회귀/개선/diff) 는 판정 전 본문 직접 인용 — "본문 표에 `+366% 개선` 으로 표기됨" 같이 메인 응답에 인용
- [ ] DoD 위반 판정 전 sub-agent 에 "내가 읽은 방식이 맞는지" 1회 체크 메시지 (선택)

##### SSoT JSON 필드 설계 규약 (제안)

- `*_pct` 는 **절대값만** 표시
- 부호 의미는 별도 필드로 분리: 예 `D4_direction`: `"improvement"` | `"regression"`
- 필드명만으론 규약 추론 불가하게 설계 — 필드명에 의미 단어를 박지 않거나, 박을 때는 부호 규약을 별도 메타 필드로 명시

#### 관련 교훈

- CLAUDE.md "수치 DoD 미달 시 측정 방법 검증 우선" (사용자 원칙 #10)
- CLAUDE.md "AI가 자기 작업을 과도하게 평가하는 경향"
- "빌드 성공 ≠ 동작하는 앱" (volt 기존)
- "sub-agent 검증 완료 ≠ GitHub 박제 완료" (volt #24) 의 메인 버전 — 자기 과대평가 방향만 반대

## 관련 노트/링크

- astro-simulator PR [#322](https://github.com/coseo12/astro-simulator/pull/322) (P11-B.2 bench — 본 오판 발생 PR)
- astro-simulator 이슈 [#289](https://github.com/coseo12/astro-simulator/issues/289) (P11-B)
- 리포트 본문 경로: `docs/benchmarks/p11-b-bench-20260424.md` (양수=개선 명확 표기)
- CLAUDE.md 스프린트 계약 #10 "수치 DoD 미달 시 측정 방법 검증 우선"
