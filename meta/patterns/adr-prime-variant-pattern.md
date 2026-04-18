---
title: ADR 결정 vs 구현 기술 장벽 — D' 변형(prime) 박제 패턴
type: pattern
source_repo: coseo12/astro-simulator
source_issue: 26
captured_at: 2026-04-18
status: refined
tags: [adr, architecture-decision, design-evolution, implementation-reality, retrospective-hygiene, architect-persona]
related:
  - ../../notes/babylon-postprocess-alpha-silent-failure.md
  - ../../notes/swiftshader-headless-pixel-freeze.md
---

## 배경/상황

astro-simulator P6-B (accretion disk + 블랙홀 shadow 파이프라인) 구현 중, architect 페르소나가 박제한 ADR 결정과 실제 구현 사이에 **기술 장벽에 의한 불일치**가 발생했다.

**ADR 원안 (1)-D**: 광선별 3D ray construction (WGSL invViewProj 역행렬) + LUT 샘플링
**실제 구현**: 화면공간 b/Rs 근사 + LUT 샘플링 (P5-D 기존 패턴 확장)

**원인**: Babylon WebGPU의 `setMatrix` invViewProj 행렬 전달 방식으로 3D ray 재구성 시 fragment shader가 검은 화면을 silently 출력 (관련 볼트 노트: `notes/babylon-postprocess-alpha-silent-failure.md`, 원본 이슈 #25). uniform 추가만으로도 회귀 재현.

**사용자 결정**: 옵션 제시 — (A) ADR 정정 + 그대로 PR / (B) invViewProj 재시도 / (C) 범위 축소 — 사용자는 **A+B 동시 진행** 선택.

## 내용

**채택한 박제 방식 — ADR "변형(D' prime)" 표기**:

1. **원안과 변형의 명시적 구분** — ADR `결정` 섹션 (1)-D 밑에 "**D' 변형**" 서브섹션 추가:
   - 변경 이유: Babylon invViewProj 처리 이슈
   - 본질적 영향: shadow LUT 정확성/UI 동적성/fps 모두 만족. 광선 매핑 방식만 변경.
   - 차이점: 3D 광선 경로 → 화면공간 매핑
2. **재검토 트리거 추가**: "invViewProj 처리 해결 시 D 원안 복원 — 트랙 B에서 재시도 예정"
3. **후속 이슈 분리**: 별도 fix 이슈(#196)로 D 원안 복원 작업 추적 + 추가 디버깅 단서(시도한 옵션 / 스크린샷 / 추정 원인) 박제
4. **PR 본문에 명시적 경고**: "⚠️ ADR 변경 명시" 섹션 — reviewer가 변형을 인지하고 판단할 수 있도록

**대안 고려**:

- ADR 폐기 → 정보 손실
- ADR 본문 통째 재작성 → 원안 의도 소실, 후속 복원 어려움
- 별도 ADR 신규 작성 → 의사결정 이력 단절

**검증**:

- reviewer가 정적 리뷰에서 "D' 변형 박제 충실" 판정 — 원안 의도/변경 사유/본질 영향/재검토 트리거 모두 검증
- QA가 DoD 본질적 만족 확인 (shadow LUT 정확성 / UI 동적성 / fps)

## 교훈 / 다음에 적용할 점

1. **architect 결정과 구현 현실의 불일치는 drop/rewrite보다 "변형 박제"가 적합** — 원안 의도를 유지하면서 현실 조정 투명 기록. reviewer·qa·후속 작업자가 차이를 인지하고 판단 가능.

2. **ADR 스키마에 "변형" 확장 여지를 기본 포함**:
   - `결정` 섹션 밑에 `결정 (YYYY-MM-DD 변형)` 서브섹션 추가 가능한 구조
   - 원안 결정은 **절대 편집하지 않음** (deletion/rewrite 금지) — 이력 보존
   - 변형 서브섹션에 `변경 이유` / `본질 영향` / `원안 대비 차이` / `원안 복원 트리거` 4항목 필수

3. **기술 장벽으로 결정 변경 시 fix 이슈 스핀아웃** — 후속 복원 작업을 명시적 추적. fix 이슈에는 시도한 옵션 / 실패 스크린샷 / 추정 원인을 박제해 다음 작업자의 재시작 비용 최소화.

4. **PR 본문에 "ADR 변경 명시" 경고 섹션 필수** — 머지 후 ADR이 원안과 다른 상태로 박제됨을 reviewer/후속 작업자가 즉시 인지 가능. 머지된 ADR을 그대로 믿으면 안 된다는 신호.

5. **사용자 결정을 강제 요청**: architect 결정 이탈 발견 시 자동 폐기/진행 금지, 사용자 옵션 제시(A/B/C) 후 명시적 승인 필수. AI sub-agent가 임의로 결정 변경하면 의사결정 이력이 흐려진다.

6. **"변형"은 "실패"가 아니다** — ADR 방법론 관점에서 변형 박제는 설계 학습의 기록. 원안을 복원 가능한 상태로 유지하면서 현실 조정을 투명 기록하는 것이 숨기거나 폐기하는 것보다 장기적으로 가치 있음.
