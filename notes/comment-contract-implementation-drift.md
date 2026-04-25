---
title: 주석 계약 vs 구현 drift 자체가 버그 생성원 — flaky test 원인 사례
type: knowledge
source_repo: coseo12/harness-setting
source_issue: 49
captured_at: 2026-04-20
status: refined
tags: [documentation, contract, implementation, drift, flaky, categorization, testing]
related:
  - ../meta/patterns/sub-agent-finalization-miss-pattern.md
  - ./ci-green-no-test-execution-trap.md
  - ./flaky-test-suppress-vs-root-cause.md
  - ./ci-multi-language-design-4-principles.md
  - ./debug-script-over-static-analysis.md
  - ./hidden-constant-proportion-drift.md
  - ./monorepo-core-dist-stale.md
  - ./roi-test-skip-judgment-drift.md
---

## 요약

파일 상단 주석이 선언한 **계약** (예: "X 카테고리는 Y 규칙 포함") 이 구현에 반영되지 않은 상태 (주석-구현 drift) 는 **버그 생성원**. 이번 사례에서는 `categorize.js` 의 "user-only: state, **logs**, 사용자 추가 파일" 주석이 선언됐지만 `.claude/logs/` 규칙이 구현에 없어 `atomic` 으로 오분류되었고, 병렬 테스트 75% 실패율의 직접 원인이 되었다.

## 본문

#### 관찰 사례

```js
// lib/categorize.js 상단 주석
/**
 *   - user-only     : init 후엔 harness가 손대지 않음 (state, logs, 사용자 추가 파일).
 */
function categorize(relPath) {
  if (p.startsWith('.harness/')) return 'user-only';
  if (p === '.gitignore') return 'user-only';
  // .claude/logs/ 규칙 ⚠️ 누락 — 주석 계약과 구현이 어긋남
  ...
  return 'atomic';  // fallback
}
```

- `.claude/logs/cross-validate-*.log` 338 개 (세션 누적) 가 `atomic` 으로 fallback
- `walkTracked(PKG_ROOT)` 가 이들을 tracked 에 포함 → `update(cwd, {applyAllSafe})` 가 매번 copy 대상으로 삼음
- post-apply 검증 단계의 해시 재계산이 병렬 I/O 경쟁과 timing 으로 일부 파일에서 불일치 → `rolledBack` 배열 비어있지 않음 → `result.ok=false`
- 병렬 테스트 `post-apply 검증: 정상 apply 시 ok=true` 약 75% 빈도 실패

#### 수정

**1 라인 추가** — 주석 계약을 구현에 반영:

```js
if (p.startsWith('.claude/logs/')) return 'user-only';
```

효과:
- 8회 연속 병렬 실행 실패 0건
- 로컬 18s (sequential 임시 조치) → 7.5s (parallel)
- CI 15s → 2.4s

#### 범용화

- **"주석에 선언된 규칙 / 계약 / 불변식" 은 테스트 커버리지 대상** — 주석으로 명시한 동작은 자동 검증되어야 한다. 주석만 있고 구현이 누락되면 드리프트 시 조용히 bug
- **리팩토링 시 주석-구현 일치 감사**: 주석에 나열된 항목을 함수 시그니처/분기와 대조. 누락 발견 시 "주석이 틀렸는가, 구현이 틀렸는가" 판정 후 일치
- **카테고리 / enum 류 분기 함수 특히 주의**: default fallback 이 존재하는 패턴 (`return 'atomic'`) 은 누락을 조용히 흡수. fallback 에 `console.warn` 또는 테스트에서 예상 카테고리 assert 로 드리프트 감지
- **리팩토링 vs 버그 수정의 경계**: 주석 계약에 구현을 맞추는 것은 엄밀히는 **버그 수정** (주석이 이미 계약). 그러나 downstream 영향 (tracked 파일 세트 변화) 이 있으면 릴리스 분류는 **MINOR**

#### 진단 루트 (재사용 가능)

1. 실패 stderr/로그에서 rolledBack / 영향받은 객체 특정
2. 해당 객체의 카테고리 / 분기 결과 확인 (`console.log(categorize(...))`)
3. 분기 함수의 상단 주석 계약과 대조
4. 드리프트 발견 시 주석이 정본인지, 구현이 정본인지 판정
5. 수정 + 회귀 가드 (가능하면 주석 계약을 자동 검증하는 테스트)

#### 인접 원칙

- CLAUDE.md `실전 교훈` "빌드 성공 ≠ 동작" 의 구현 버전 — 주석 읽는 것 ≠ 구현 확인
- volt [#24](https://github.com/coseo12/volt/issues/24) "sub-agent 검증 완료 ≠ 박제 완료" 와 병렬 — 선언 측 (주석/보고) 과 실행 측 (구현/박제) 간 간극 이라는 상위 카테고리
