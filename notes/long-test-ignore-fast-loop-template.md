---
title: Rust 장기 테스트 #[ignore] 분리 규범 — 일상 cargo test 5분 내 완주 템플릿
type: knowledge
source_repo: coseo12/astro-simulator
source_issue: 54
captured_at: 2026-04-20
status: refined
tags: [rust, cargo, ci-cd, test-infrastructure, ignore-attribute, workflow-pattern, harness-rag]
related:
  - ../meta/patterns/sub-agent-bg-process-leak-pattern.md
---

## 요약

장기 적분·E2E·외부 API 테스트가 일상 `cargo test --lib` 경로에 혼재하면 TDD 루프가 교착된다 (astro-simulator 에서 30분+ 관찰). `#[ignore = "<사유>; run with --include-ignored in CI"]` 어트리뷰트로 분리 + CI `--include-ignored` 독립 job (`continue-on-error: true`) 구성으로 **일상 경로 5분 내 완주** 를 보장한다. astro-simulator P9 M4 실측: 30분+ 교착 → 9.27s (200× 단축, 32× 여유). Jest/Vitest/pytest 등 타 언어 확장 규범도 함께 박제.

## 본문

#### 문제 현상

Rust 프로젝트에서 `cargo test --lib` 경로가 다음 요인으로 수 분 ~ 수 시간 지연되는 사례:

- **장기 적분 테스트**: 수치 시뮬레이터의 세차 / 궤도역행 검증 (100+ 주기 적분)
- **E2E 통합 테스트**: 전체 파이프라인 경유 검증
- **외부 리소스 호출**: 느린 네트워크 / 파일 I/O / DB 시드
- **stress / property-based**: 1000+ 반복 랜덤 케이스

이런 테스트가 **빠른 단위 테스트와 혼재**하면 TDD 루프가 무너진다. 개발자가 로컬에서 `cargo test` 를 돌리기 싫어지고, CI 시간도 예측 불가능해진다.

#### 해결 규범: `#[ignore]` + CI `--include-ignored` 이원화

**Rust 테스트 마킹**:
```rust
#[test]
#[ignore = "long-integration; run with --include-ignored in CI"]
fn mercury_perihelion_precession_eih() {
    // 100+ 년 적분
}
```

- 사유 문자열 필수 — 재현 조건 박제
- 규약 문구 통일 (`long-integration` / `e2e` / `requires-network` 등) — grep 검색 용이

**CI 워크플로 이원화** (GitHub Actions 예시):

```yaml
jobs:
  test-fast:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: cargo test --release --lib
    # 실패 시 PR 차단

  test-long-integration:
    runs-on: ubuntu-latest
    continue-on-error: true  # 빠른 경로 실패 시만 머지 차단
    steps:
      - uses: actions/checkout@v4
      - run: cargo test --release --lib -- --include-ignored
    # 실패는 nightly 알림 / 대시보드만
```

**핵심**: 두 job 이 **독립 캐시 키** 사용 — 빠른 경로 재실행이 장기 경로 cache 를 무효화하지 않음.

#### 실측 사례 (astro-simulator P9)

- **대상 6건 `#[ignore]` 적용**:
  - `mercury_perihelion_precession_eih`
  - `yoshida_mercury_perihelion_regression`
  - `earth_perihelion_eih_within_5_percent`
  - `yoshida_earth_perihelion_regression`
  - `venus_perihelion_eih_within_5_percent`
  - `yoshida_venus_perihelion_regression`
- **일상 `cargo test --lib`**: 30분+ 교착 → **9.27s** (200× 단축, 목표 5분 32× 여유)
- **CI `--include-ignored`**: 216.9s (3분 36초, `continue-on-error: true`)

#### 타 언어 확장 패턴

| 언어/프레임워크    | 마킹                                    | 실행 전환                          |
| ------------------ | --------------------------------------- | ---------------------------------- |
| Rust               | `#[ignore = "..."]`                     | `cargo test -- --include-ignored`  |
| Jest/Vitest        | `test.todo` / `describe.skip`           | `vitest --grep "<tag>"`            |
| Jest (수동 sel)    | `it.extend.only` + env flag             | `RUN_SLOW=1 jest`                  |
| pytest             | `@pytest.mark.slow`                     | `pytest --run-slow` (conftest hook) |
| Go                 | `t.Skip(...)` + build tag `//go:build slow` | `go test -tags slow ./...`         |
| JUnit (Java)       | `@Tag("slow")` + Maven `-DexcludedGroups=slow` | Maven profile 전환            |

#### 재발 감지 신호

- 로컬 `cargo test` 가 이전 대비 **5배+ 소요**
- CI 빠른 경로 캐시 무효화 빈도 증가
- 개발자가 `--lib` 없이 개별 테스트만 실행하는 습관 (빠른 경로 신뢰 저하)
- target/debug/ 디렉토리 크기 폭증 (동시 test binary 누적 — astro-simulator [volt #52](https://github.com/coseo12/volt/issues/52) 참조)

#### 주의사항

- **`#[ignore]` 남용 경계** — 단순히 "불편한 테스트" 를 숨기는 용도로 쓰면 장기 경로가 커버리지 블랙홀이 됨. 사유 문자열 + CI 실측 시간 기록 필수
- **CI 장기 경로 모니터링** — `continue-on-error: true` 는 빠른 경로 보호일 뿐. 장기 실패를 오래 방치하지 않도록 dashboard / nightly 알림 필수
- **cross-compile 경로**: `#[cfg(not(target_arch = "wasm32"))]` 등과 조합 시 CI matrix 확장 영향 확인

## 관련 노트/링크

- [astro-simulator P9 회고 §M4 섹션](https://github.com/coseo12/astro-simulator/blob/main/docs/retrospectives/p9-retrospective.md) — 실측 근거
- [astro-simulator #260 (P9 PR-2)](https://github.com/coseo12/astro-simulator/pull/260) — M4 구현 원천
- [volt #52 sub-agent 프로세스 리크](https://github.com/coseo12/volt/issues/52) — 좀비 누적 원인 (M4 의 배경 동기)
- [CHANGELOG [0.9.0] §Behavior Changes — M4 장기 테스트 분리](https://github.com/coseo12/astro-simulator/blob/main/CHANGELOG.md)
