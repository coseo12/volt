---
title: harness-setting 의 pnpm workspace + WASM 모노레포 스모크 테스트 부재
type: knowledge
source_repo: coseo12/harness-setting
source_issue: 64
captured_at: 2026-04-20
status: refined
tags: [harness, ci, pnpm, workspace, wasm, smoke-test, upstream-coverage]
related:
  - ./downstream-structure-compat-checklist.md
  - ./session-multi-release-drift-signal.md
  - ../meta/patterns/dogfooding-guard-self-violation-pattern.md
  - ./downstream-measurement-final-guard.md
---

## 요약

harness-setting 자체가 npm 단일 프로젝트라 다운스트림에서 발견되는 pnpm workspace / WASM 모노레포 구조와의 부합 버그가 **upstream CI 에서 드러나지 않음**. 2026-04-20 세션에서 v2.28.2 → v2.29.1 에 걸쳐 pnpm 관련 연쇄 버그 3건 관찰. upstream CI 에 대표 구조 스모크 테스트 fixture 추가 제안.

## 본문

#### 배경

v2.15.0 에서 `detect-and-test` 가 "감지만" 에서 "실제 실행" 으로 확장 (#153). upstream 은 npm 프로젝트라 **npm 경로만 검증** 됨. pnpm 프로젝트 확산에도 스모크 테스트 부재.

#### 관찰된 버그 연쇄 (2026-04-20)

| 버전 | 버그 | 영향 | 발견 경로 |
|---|---|---|---|
| v2.28.2 (#176) | pnpm 경로 부재 — `sh: 1: pnpm: not found` | pnpm 프로젝트 CI 전면 실패 | 다운스트림 astro-simulator#270 |
| v2.28.2 도입 직후 | `pnpm test --if-present` forwarding → vitest "Unknown option" | pnpm 모노레포 CI 실패 | 다운스트림 재실험 (v2.29.1 #182 수정) |
| v2.29.1 이후 | WASM 모노레포 (core → physics-wasm 타입 의존) 구조 부합 불가 | harness 범용 CI 로 해결 불가 | 다운스트림 scripts.test 제거로 회피 |

각 버그가 upstream CI 에서 **사전에 드러날 수 있었음** — pnpm workspace fixture 가 있었다면.

#### 제안: upstream CI fixture

**1. 기본 pnpm workspace fixture** (`test/fixtures/pnpm-monorepo/`)

```
test/fixtures/pnpm-monorepo/
├── package.json              # packageManager: pnpm@X, scripts.test: pnpm -r test
├── pnpm-workspace.yaml       # packages: [packages/*]
├── pnpm-lock.yaml
└── packages/
    ├── lib/
    │   ├── package.json      # exports: ./dist/..., scripts.build/test
    │   ├── src/index.ts
    │   └── vitest.config.ts
    └── app/
        ├── package.json      # depends on @fixture/lib
        └── src/index.test.ts
```

- `@fixture/lib` 이 **dist-based exports** 로 `@fixture/app` 에서 import
- `pnpm install && pnpm test` 가 통과해야 pass
- 이런 fixture 가 있었으면 `pnpm test --if-present` forwarding 버그 즉시 감지

**2. 확장 WASM fixture** (선택, opt-out 기본)

- `test/fixtures/pnpm-wasm-monorepo/` — core → physics-wasm 타입 의존 재현
- 범용 CI 가 감당 불가한 구조임을 **명시적 문서** 에 선언 ("harness 는 wasm-pack 미지원 — 프로젝트 전용 워크플로로 분리 권고")
- 이 fixture 는 detect-and-test 가 **의도적으로 skip** 하는 경로 (scripts.test 부재) 를 검증

**3. upstream CI 통합**

harness-setting 의 `detect-and-test` 자체 워크플로에 "fixture 디렉토리에 ci.yml 을 복사 후 fixture 내부에서 실행" step 추가. 릴리스 전 필수 게이트.

#### 기대 효과

- 새 릴리스가 pnpm/WASM 다운스트림에서 깨지는 케이스를 **upstream 단계에서 사전 감지**
- 다운스트림 (astro-simulator 같은 프로젝트) 의 push-fail-fix 6회 루프 제거
- harness-setting 의 `### Behavior Changes` 섹션에 "pnpm/WASM 호환성 보증" 을 명시적으로 주장 가능

#### 본 knowledge 가 해결하지 않는 것

- harness 가 **범용 npm 기반** 이라는 근본 설계는 유지. pnpm 전용 기능 (workspace 필터 / 특수 플래그) 을 harness 가 수용하려면 별도 범위 논쟁 필요
- WASM / Cargo / protoc 등 특수 빌드 도구는 **여전히 프로젝트 전용 워크플로 몫** — fixture 는 이를 "감당 불가 경계" 로 박제하는 용도

## 관련 노트/링크

- harness-setting PR #176 (v2.28.2), PR #182 (v2.29.1), 이슈 #175, #181
- 다운스트림 관찰: astro-simulator PR #270 (6 커밋 누적)
- 근거 세션: 2026-04-20 P10-A 착수 세션
- 연관 volt: #62 (범용 CI 템플릿 부합성 체크리스트), #63 (세션 이탈 시그널)
