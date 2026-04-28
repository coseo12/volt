---
title: harness 범용 CI 템플릿과 다운스트림 구조 부합성 사전 점검 체크리스트
type: knowledge
source_repo: coseo12/astro-simulator
source_issue: 62
captured_at: 2026-04-20
status: refined
tags: [harness, ci, pnpm, monorepo, wasm, compatibility-check]
related:
  - ../meta/patterns/dogfooding-guard-self-violation-pattern.md
  - ./downstream-measurement-final-guard.md
  - ./pnpm-wasm-monorepo-upstream-fixture.md
  - ./regression-guard-bootstrap-chicken-and-egg.md
---

## 요약

harness update 적용 직후 CI 가 다운스트림 프로젝트 구조 (pnpm 모노레포 + WASM 타입 의존 + dist-based exports) 와 부합하지 않아 6회 연속 push-fail-fix 루프에 빠진 사례. harness `--check` 후 구조 부합성을 확인하는 체크리스트가 있었으면 즉시 "scripts.test 제거" 전략을 도출 가능.

## 본문

#### 배경

astro-simulator 에서 harness v2.15.0 → v2.28.1 적용 후 `detect-and-test` job 실패. v2.15.0 (#153) 에서 `detect-and-test` 가 "감지만" 에서 "실제 npm test 실행" 으로 확장됐을 때 pnpm 경로·WASM 의존성 등 다운스트림 특수 구조와의 부합 여부를 사전 검증할 체계가 없었음.

#### 관찰된 실패 연쇄

1. `npm test` 실행 → `sh: 1: pnpm: not found` → pnpm 경로 없음 (harness-setting v2.28.2 수정)
2. `pnpm test --if-present` → vitest "Unknown option" → `--if-present` forwarding 버그 (v2.29.1 수정)
3. `pnpm -r test` → physics-wasm 의 `wasm-pack not found` → 범용 CI 는 wasm-pack 미설치
4. physics-wasm 제외 → shared dist 없음 → build 선행 필요
5. `pnpm -r build` 필터 → root.build 재귀로 physics-wasm 다시 포함
6. core build → `Cannot find module @astro-simulator/physics-wasm` → 타입 수준 의존

6번째 실패에서야 구조 충돌을 인지하고 "scripts.test 제거" 로 선회. 처음부터 구조 부합성을 점검했다면 단축 가능.

#### 체크리스트 제안

harness update 전 `--check` 직후 수행:

- [ ] 다운스트림 `package.json scripts.test` 가 **모노레포 전체 재귀 호출** 인가? (`pnpm -r` / `nx run-many` / `turbo run test` 등)
  - Yes → CI 가 test 호출 시 **모든 workspace** 가 빌드·테스트에 참여함을 인지
- [ ] 테스트 대상 중 **빌드 산출물 기반 exports** (dist/main/module) 의존이 있는가?
  - Yes → `pnpm install` 직후 import 불가 → 빌드 선행 필요 → detect-and-test 의 설치→테스트 단순 파이프라인 부적합
- [ ] 워크스페이스 중 **특수 빌드 도구** (wasm-pack / cargo / protoc / msbuild / bindgen) 의존이 있는가?
  - Yes → 범용 Node CI 템플릿은 미지원 → 해당 workspace 제외 또는 사전 설치 step 필요
- [ ] `verify-and-rust` / `verify-wasm` / 프로젝트 전용 테스트 워크플로가 **이미 전체 테스트를 수행** 하고 있는가?
  - Yes → harness detect-and-test 는 **중복** → 프로젝트에서 scripts.test 제거로 detect-and-test 자동 skip 권고

#### 구조 불일치 시 선택 옵션

| 옵션 | 내용 | 트레이드오프 |
|---|---|---|
| A | `scripts.test` 제거 | `pnpm test` 로컬 관습 변경, detect-and-test skip |
| B | `scripts.test` 를 no-op shim | 의도 명시, 하지만 "fake" 스크립트 |
| C | ci.yml 을 divergent 수정 | upstream 과 영구 충돌 |
| D | harness upstream 확장 | 왕복 비용, 범용성 논쟁 |

판단 애매 시 **A** 추천 — `pnpm test` 대신 `test:unit` / `test:ci` 로 명시 호출 권장.

## 관련 노트/링크

- astro-simulator PR #270 (harness v2.15→v2.29.1 적용, 6 커밋 누적)
- harness-setting PR #176 (v2.28.2 — pnpm 경로 도입), PR #182 (v2.29.1 — --if-present 수정)
- 근거 세션: 2026-04-20 P10-A 착수 세션
