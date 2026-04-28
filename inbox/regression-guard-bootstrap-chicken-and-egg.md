---
title: 회귀 가드 인프라 chicken-and-egg + workspace binary 의존 (#60 추가 사례)
type: report
source_repo: coseo12/astro-simulator
source_issue: 78
captured_at: 2026-04-28
status: inbox
tags: [chicken-and-egg, workspace-binary-dependency, monorepo, ci-infrastructure, ux-regression-guard, downstream-verification, retrospective, upstream-blindspot]
related:
  - ../notes/downstream-measurement-final-guard.md
  - ../notes/downstream-structure-compat-checklist.md
  - ../notes/pnpm-wasm-monorepo-upstream-fixture.md
---

## 배경/상황

`coseo12/astro-simulator` PR [#347](https://github.com/coseo12/astro-simulator/pull/347) (이슈 #337 R1 후속 F-1) — R1 UI 회귀 가드 baseline (4 영역 × 3 viewport = 12 PNG, pixelmatch threshold=0.1, mismatch ≤ 0.5%) 을 macOS 캡처본에서 CI Linux 캡처본으로 전환하는 부트스트래핑 인프라 도입.

architect → developer → reviewer 3단계 정적 방어를 모두 통과한 1차 구현 (commit `eadce1e`) 이 다운스트림 CI 실측에서 두 가지 잠재 fail 동시 표면화:

1. `wasm-pack: not found` — `detect-and-test` job 의 r1-guard step 의 `pnpm build` 가 monorepo recursive 로 `physics-wasm` 워크스페이스 wasm-pack 호출
2. **잠재 chicken-and-egg** (wasm-pack 해결 후 발견 예상) — macOS 캡처 baseline 으로 Linux CI 검증 → mismatch 초과 fail

5 commit trace 박제: `dac2874` (architect ADR Amendment) → `eadce1e` (developer 1차 구현) → `70324b8` (메인 ci.yml step 분리) → `d687072` (reviewer BLOCK fix) → `c3e951c` (ADR Amendment v2 박제).

## 내용

3 패턴이 단일 사이클에서 직교적으로 발견:

#### 패턴 1 — 회귀 가드 인프라 chicken-and-egg

**사실관계**: 회귀 가드 인프라 (`pixel diff baseline + step + 임계값`) 도입 PR 이 baseline + step 통합을 동일 PR 에 묶으면 머지 시점 baseline 정합성 미확보 → step 자명 fail.

**현상 메커니즘**:
- 부트스트래핑 절차 (a) PR 머지 → (b) `workflow_dispatch` → (c) baseline 갱신 PR 머지 가정
- (a) 단계 PR 자체가 `ci.yml` r1-guard step 추가 → 머지 시점 baseline 은 여전히 OS 미정합 캡처본 → step fail
- 즉 (a) 단계의 step 자체가 chicken 위치

**해결 패턴**: PR 분리 — workflow + 매개변수화만 PR1 머지 → 부트스트래핑 (b)~(c) → ci.yml step 통합 PR2.

**판정 질문 (사전 진단 체크리스트 박제)**:
> "본 PR 의 step 이 동시에 추가하는 baseline 의 OS/환경 정합성 확인 안 된 시점에 동작 가능한가?"

#### 패턴 2 — monorepo recursive build 의 workspace binary 의존

**사실관계**: `pnpm build` (root) ≠ `pnpm --filter <pkg> build`. recursive 가 다른 워크스페이스의 binary 의존 (wasm-pack, cargo, rustc, protoc 등) 을 끌어옴. CI job 별로 binary 매트릭스가 다르면 새 step 도입 시 즉시 fail.

**의존 그래프 (실측)**:
```
apps/web (next build) → packages/core → packages/physics-wasm (wasm-pack build:node + build:bundler)
```

`physics-wasm` 워크스페이스 build 스크립트가 `wasm-pack build --target nodejs` + `--target bundler` 호출. `verify-and-rust` job 만 wasm-pack 보유 (`taiki-e/install-action`); `detect-and-test` job 은 미설치.

**해결 패턴 (검증된 3 step)**:
```yaml
- uses: dtolnay/rust-toolchain@stable
  with:
    toolchain: '1.94.1'
    targets: wasm32-unknown-unknown
- uses: Swatinem/rust-cache@v2
  with:
    workspaces: packages/physics-wasm
- uses: taiki-e/install-action@v2
  with:
    tool: wasm-pack@0.14.0
```

**사전 진단 체크리스트 박제** (관련: 본 레포 #64):
- 새 workflow/job 도입 전 빌드 트리 binary 의존 매트릭스 박제 의무
- 추적 명령:
  ```bash
  pnpm --filter <target-pkg>... why <suspected-pkg>
  grep -rn '"build":' packages/*/package.json apps/*/package.json
  ```
- 박제 형식 예: `web → core → physics-wasm → wasm-pack ⊃ rustc`

#### 패턴 3 — upstream 3중 방어 blindspot 추가 사례 (본 레포 #60)

본 레포 [#60](https://github.com/coseo12/volt/issues/60) 이 박제한 "다운스트림 실측이 최종 가드" 의 구체 사례 1건 추가:

**3중 방어 통과 + 다운스트림 fail**:

| 단계 | 검증 항목 | 결과 | blindspot |
|------|-----------|------|-----------|
| architect ADR Amendment (`dac2874`) | 후보 B (Linux baseline) + workflow 분리 + Concrete Prediction | 통과 | wasm-pack 의존 / chicken-and-egg 미짚음 |
| developer 1차 구현 (`eadce1e`) | macOS 로컬 검증 (`pnpm start -p 3001` HTTP 200, `SKIP_LOCAL=1 + macOS` 즉시 PASS, 단위 테스트 441 PASS) | 통과 | local ≠ CI Linux 환경 차이 |
| reviewer 정적 분석 (1차) | 5축 검증 | 통과 | 자체 workflow yaml 의 wasm-pack 누락 미발견 (단, 메인이 ci.yml step fail 후 호출한 reviewer 2차 라운드에서 발견 — 즉 다운스트림 fail 이 reviewer 검토 영역 확장 트리거 역할) |
| 다운스트림 CI 실측 | `detect-and-test` FAIL | **`wasm-pack: not found` 박제** | — |

**메인 오케스트레이터의 발견 가치**: developer self-compare 자명 PASS (CLAUDE.md `### sub-agent 검증 완료 ≠ GitHub 박제 완료`) 를 다운스트림 CI 실측이 차단. 이후 메인이 reviewer 재호출 시 reviewer 가 자체 workflow 의 wasm-pack 누락 (BLOCK-1) 까지 발견 — **다운스트림 fail 이 정적 검증 영역 확장 트리거 역할** 이라는 추가 관찰.

## 교훈 / 다음에 적용할 점

#### Agent / Skill 개선 후보

1. **architect 페르소나 사전 진단 체크리스트 추가** (`.claude/agents/architect.md`):
   - 회귀 가드 인프라 도입 시 chicken-and-egg 인지 의무 (baseline + step 통합 시점 정합성)
   - monorepo recursive build 의 binary 의존 매트릭스 박제 의무 (workflow/job 별)
   - 부트스트래핑 절차 (a)~(e) 박제 시 각 단계의 잠재 fail 명시
2. **reviewer 페르소나 5축 + binary 의존성 축 추가** (`.claude/agents/reviewer.md`):
   - 신규 workflow yaml 의 step 별 binary 의존 검증
   - 의존 그래프 추적 명령 (위 §패턴 2) 적용
3. **ADR record-adr 스킬 템플릿에 §"잠재 fail 시나리오" 섹션 추가** (`.claude/skills/record-adr/SKILL.md`):
   - 결정 시점에 후속 발견 가능한 fail 패턴 미리 박제 (자기 ADR 의 한계 인지)

#### Harness 패턴 박제 권고

- 본 레포 [#60](https://github.com/coseo12/volt/issues/60) 의 "다운스트림 실측이 최종 가드" 에 **반복 관찰 횟수 3 → 4** 로 카운트 갱신 권고 (CLAUDE.md `### 다운스트림 실측이 최종 가드` 섹션 박제)
- 본 레포 [#64](https://github.com/coseo12/volt/issues/64) 의 "harness-setting pnpm workspace + WASM 모노레포 스모크 테스트 부재" 에 본 사례 추가 — 다운스트림 사용자 (astro-simulator) 도 동일 패턴 재발견. harness upstream 의 fixture 추가 우선순위 상승 권고

#### 향후 R-Phase 적용 (R2~R10)

본 사이클 발견은 향후 행성별 가시성 R-Phase (R2 수성, R3 금성, ..., R10) 의 회귀 가드 인프라 도입 시 동일 패턴 재발 위험. ADR `20260425-r1-ui-pixel-diff-guard.md` §Amendment v2 §사전 진단 체크리스트 (3건) 가 박제됐으나, 매 phase 마다 적용 여부 점검 의무.

#### 참고 링크

- 본 레포 #60 (3중 방어 blindspot): https://github.com/coseo12/volt/issues/60
- 본 레포 #62 (CI 다운스트림 부합성): https://github.com/coseo12/volt/issues/62
- 본 레포 #64 (pnpm workspace + WASM 부재): https://github.com/coseo12/volt/issues/64
- ADR Amendment v2: https://github.com/coseo12/astro-simulator/blob/develop/docs/decisions/20260425-r1-ui-pixel-diff-guard.md (§"Amendment v2 2026-04-26")
- PR #347: https://github.com/coseo12/astro-simulator/pull/347
- 후속 이슈 #348: https://github.com/coseo12/astro-simulator/issues/348
