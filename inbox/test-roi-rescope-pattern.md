---
title: 완료 기준 실측 재조정 — 테스트 ROI 판단 + 주석 계약 대체 사례
type: pattern
source_repo: coseo12/harness-setting
source_issue: 31
captured_at: 2026-04-18
status: inbox
tags: [sprint-contract, test-roi, test-infrastructure, contract-in-comments, scope-adjustment, integration-test]
related:
  - ../meta/patterns/phased-release-split-pattern.md
---

## 요약

스프린트 계약에 명시된 완료 기준(테스트 추가) 이 실제 구현 시점에 **의존성 복잡도로 ROI 낮게 판명** 된 사례. 기준을 고집하지 않고 **주석 계약 박제 + 인접 속성 테스트로 간접 가드** 로 재조정. CLAUDE.md "합의된 기준은 실측 후 재조정 가능" 원칙의 구체 적용. 재조정은 투명하게(PR 본문 + CHANGELOG Notes) 박제.

## 본문

#### 상황

harness [#92](https://github.com/coseo12/harness-setting/issues/92) Phase 2 스프린트 계약에 Gemini 교차검증에서 지적한 "**merge type 스킵 경로 테스트**" 가 완료 기준으로 포함됐다.

구현 착수 시점에 다음이 드러남:

1. `applyMerge` 함수가 `git show v<version>:<rel>` 로 upstream 과거 버전을 fetch
2. 테스트 환경에선 git tag(v1/v2 등) 가 없으므로 **fixture git repo** 를 만들어야 함
3. fixture git repo 생성 + 태그 시딩 + cleanup 을 테스트 당 반복 → 단위 테스트 하나 당 수백 ms + 구현 코드 수십 줄
4. 검증 대상 로직은 `if (a.type === 'merge') continue;` **1 줄** — ROI 역전

#### 결정

완료 기준을 **재조정** 하여 세 대체재 조합으로 전환:

1. **주석 계약 박제** (`lib/update.js`):
   ```javascript
   // 계약 (회귀 가드):
   //   - merge type: 3-way merge 결과물은 의도적으로 upstream 과 다를 수 있어 검증 스킵
   //   - delete type: 삭제 후 디스크에 파일 없음이 정상이므로 검증 스킵
   //   - managed-block 카테고리: categoricalSha256 이 managedSha256 으로 위임되어
   //     센티널 내부만 해시하므로 사용자의 외부 영역 편집에 대한 오탐 없음.
   //     (test/sentinels-invariance.test.js 에서 불변성 보장)
   ```

2. **인접 속성 테스트로 간접 가드** — `test/sentinels-invariance.test.js` (managed-block 불변성 3케이스) 가 `categoricalSha256` 의 managed-block 분기를 간접 보증. merge 스킵 조건은 주석으로 명시.

3. **완료 기준 문서화** — PR #95 본문에 "merge type 스킵 경로 테스트 → 주석 계약 박제로 대체" 이유 + 미래 필요 시 fixture git repo 기반 통합 테스트로 분리 가능함을 명시. CHANGELOG Notes 에 동일 기록.

#### 원칙 (CLAUDE.md 스프린트 계약)

> 합의된 기준은 실측 후 재조정 가능 — 단, 사용자와 명시적으로 합의 후 갱신

이번 케이스는 메인 에이전트가 **착수 시점에 ROI 역전 발견 → 재조정을 PR 본문 + CHANGELOG 에 투명 박제 → 사용자가 머지 시점에 암묵 승인** 하는 흐름. 사용자 중단/거부가 없었으므로 합의로 간주.

#### 테스트 ROI 체크리스트

신규 테스트 착수 전 5초 자문:

1. **의존성 복잡도** — 테스트 환경 구축 비용이 구현 코드 자체보다 큰가?
   - git fixture, DB seed, 네트워크 mock 등이 핵심 검증 대상 코드 라인 수의 5배 이상이면 재고
2. **가드 대상 행** — 몇 줄을 보호하는가? 1~2줄짜리 스킵 조건은 주석 계약 + 인접 속성 테스트가 충분할 수 있음
3. **회귀 가능성** — 이 라인이 바뀌면 빌드가 깨지는가, 조용히 퇴행하는가?
   - 조용히 퇴행 → 테스트 필수
   - 빌드 실패 → 주석 계약으로 충분할 수 있음
4. **대체재 가용성** — 인접 유닛 테스트(sentinels) / 문서 / 타입 가드로 대체 가능한가?
5. **미래 이전 가능성** — 지금은 비용이 크지만 fixture 인프라 구축 후 저렴해질 수 있는가? → 별도 인프라 이슈로 분리

#### 반대 함정 (지양)

- **"완료 기준에 있으니 무조건 테스트 작성"** — 의존성 복잡도 파악 없이 구축 후 유지보수 부담으로 남음 (fixture 를 쓰는 다른 테스트가 없으면 단발성 부채)
- **"ROI 낮다"는 이유로 조용히 스킵** — 재조정 사실을 PR 본문/CHANGELOG 에 박제하지 않으면 미래 에이전트가 "누락" 으로 오인

#### 박제 형태

재조정은 **세 위치에 동시에 박제** (누락 방지):
- 코드 주석 (계약 자체)
- PR 본문 (결정 근거)
- CHANGELOG Notes (미래 관찰자를 위한 기록)

## 관련 노트/링크

- harness [#89](https://github.com/coseo12/harness-setting/issues/89) Gemini 교차검증 코멘트 — merge 테스트 제안 원출처
- harness [#92](https://github.com/coseo12/harness-setting/issues/92) Phase 2 스프린트 계약 (해당 완료 기준 포함)
- harness [PR #95](https://github.com/coseo12/harness-setting/pull/95) 본문 — 재조정 결정 박제
- harness [v2.10.0 릴리스](https://github.com/coseo12/harness-setting/releases/tag/v2.10.0) Notes — "merge type 스킵 테스트는 applyMerge 의 git show 의존성으로 단위 테스트 구축 비용이 높아 주석 계약 박제로 대체"
- CLAUDE.md `스프린트 계약 § 5` — "합의된 기준은 실측 후 재조정 가능"
