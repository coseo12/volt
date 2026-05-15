---
title: ADR 정정 패턴 SSoT — strikethrough + footnote + Amendment 섹션 3중 마크업
type: knowledge
source_repo: coseo12/astro-simulator
source_issue: 83
captured_at: 2026-05-06
status: inbox
tags: [adr, amendment, documentation, ssot, traceability, markdown]
related:
  - ../notes/cross-validate-accept-vs-defer.md
  - ../notes/reviewer-qa-gate-skip-headless-self-pass.md
---

## 요약

ADR 박제값 정정 시 *역사적 박제값 보존* + *정정값 즉각성* + *정정 사유 분리* 3 책임을 분리하는 패턴. `~~original~~ corrected[^id]` 본문 직접 정정 + Amendment v별도 섹션 + footnote 파일 끝 정의 + R-Phase 후속 ADR cross-link 매트릭스 (footnote 추가 vs 역사적 인용 보존 결정). reviewer 5축 정적 리뷰에서 "역사 보존 + 정정값 즉각성 + 사유 분리 3 책임 분리" 로 안정성 입증.

## 본문

#### 4 구성 요소

**1. 본문 직접 정정 (strikethrough + footnote 참조)**

```markdown
| × 75     | 213             | 3.87%             | 320              | ~~6.18%~~ 3.88%[^c1]    | 197            | 19.6%            | 평가 |
```

GitHub flavored markdown 표준 — strikethrough (`~~original~~`) 로 원본 박제값 보존 + 정정값 즉각 표시 + `[^id]` footnote 참조로 정정 사유 분리. 3개 위치 모두 동일 마크업 적용 (line 89 표 / line 98 텍스트 / line 307 회귀 가드 — 매트릭스 정합).

**2. Amendment v별도 섹션 (정정 매트릭스 + 산출식 + 인용 + 운영 영향 + Cross-validate)**

기존 Amendment v1 (라운드 1/2/3) 흐름과 v2 형식 일관. 권장 섹션:

- 발견 경위 (외부 cross-validate / 분석 근거)
- 정정 매트릭스 표 (위치 / 원본 / 정정값 / 사유)
- 산출식 재검증 (이론 / 실측 / 정합성 입증)
- 외부 인용 (Gemini 분석 등)
- 운영 영향 (현재 운영 SSoT vs 정정 대상 박제값)
- Cross-validate (박제 직후 1회, outcome / 합의 4축 / 고유 발견)
- 회귀 가드 (Amendment 의 검증 의무)
- 재검토 트리거 (기존 + 추가)
- 참조 (외부 SSoT / 이슈 / volt)

**3. Footnote 파일 끝 정의**

```markdown
[^c1]: 2026-05-06 Amendment v2 박제로 6.18% → 3.88% 정정 확정. 본 ADR §"Amendment v2 (2026-05-06)" 참조. 원본 박제값 6.18% 는 R1 본 구현 시점 (sunScale 75) 산출 오류로, R2 ADR §고유 발견 §발견 1 (Gemini 3.1 Pro Preview, 2026-04-28) 에서 발견됨. strikethrough 보존으로 원본 인용 추적 가능.
```

footnote 정의는 ADR 파일 끝 `---` 구분선 다음에 박제. GFM 렌더링 호환.

**4. R-Phase 후속 ADR cross-link 매트릭스**

후속 ADR (예: R2 ADR `20260428-r2-mercury-visualization.md`) 이 정정 대상 박제값을 인용한 위치 각각 *의도 판정*:

| 위치 | 인용 의도 | 처리 |
|------|----------|------|
| line 133 (DoD 마진 분석) | 분석 시점의 발견 | footnote `[^r1-amend-v2]` 추가 |
| line 523 (위험 미해결) | 역사적 발견 박제 | footnote `[^r1-amend-v2]` 추가 |
| line 562 (§발견 1 제목) | Gemini 발견 시점 박제 | 본문 변경 0, *역사적 컨텍스트* 보존 |
| line 564 (Gemini 발언 인용) | 발언 시점 박제 (불변) | 본문 변경 0, *역사적 컨텍스트* 보존 |
| line 568 (후속 이슈 분리 박제) | 분리 박제 사실 | "Amendment 박제 완료 (YYYY-MM-DD)" 본문 추가 + footnote |

**핵심 결정**: ADR 의 *작성 시점 스냅샷 성격* 을 보존 — Gemini 발언, 발견 시점 분석 등은 본문 변경 0 + footnote 만 삽입하여 추적성 (Traceability) 확보.

#### 단계 게이트 적용 가능 조건

코드 영향 0 (`git diff develop -- 'apps/' 'packages/' '.github/' '*.json' '*.ts' '*.tsx' '*.mjs' 'scripts/'` 출력 0) 인 ADR-only docs PR 의 경우 **qa skip 정합** (volt #77 docs only 단계 게이트 예외). 코드 변경 동반 시 풀 단계 게이트 (developer + reviewer + qa) 의무.

#### Cross-validate (박제 직후 1회) 의무

CLAUDE.md §"교차검증" §"정책·설계·ADR 박제 직후 1회 루틴" 적용. Amendment 박제 직후 Gemini 호출 — 4축 합의 (산출 정합 / 박제 위치 / cross-link 매트릭스 / NO-OP 분기 평가) 박제. outcome=applied 시 Gemini 고유 발견은 *현재 PR 범위 내* 즉시 반영, *비-범위* 는 후속 이슈 분리 (volt #29 3단 프로토콜).

#### Reviewer 안정성 입증 (5축 정적 리뷰)

reviewer 가 5축 (로직 / 보안 / 일관성 / 단순성 / 추적성) PASS + 메인 권고 3건 (footnote 렌더링 호환 / 3중 마크업 가독성 / Cross-validate 정합) PASS + 독립 산출 재현으로 박제값 정합 입증. ADR 정정 패턴 SSoT 후보로 안정성 입증.

#### NO-OP 분기 평가 (수용 vs 거부 결정)

Amendment 박제 비용 (R1 ADR 1 파일 + R2 ADR cross-link N 파일 + cross-validate 1회) vs 가치 (R-Phase 누적 시 후속 ADR 인용 신뢰도 + 후속 합류자 / Agent 인지적 부채 차단). 가치 > 비용 시 Amendment 채택, 미달 시 NO-OP ADR (별도 파일) 분기.

## 관련 노트/링크

- 사례: [astro-simulator PR #426](https://github.com/coseo12/astro-simulator/pull/426) — R1 ADR Amendment v2 박제 (sun 점유율 6.18% → 3.88%)
- 사례 ADR: [`docs/decisions/20260425-r1-sun-visualization.md` §"Amendment v2 (2026-05-06)"](https://github.com/coseo12/astro-simulator/blob/develop/docs/decisions/20260425-r1-sun-visualization.md)
- volt #29 — 고유 발견 수용 vs 후속 분리 3단 프로토콜
- volt #77 — 단계 게이트 docs only 예외
- 부모 패턴: ADR Amendment v1 (라운드 1/2/3 박제 흐름, sunScale 75 → 50 정정)
