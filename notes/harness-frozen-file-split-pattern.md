---
title: harness frozen 파일 업데이트 충돌을 파일 분리로 방지하는 패턴
type: knowledge
source_repo: coseo12/astro-simulator
source_issue: 12
captured_at: 2026-04-15
status: refined
tags: [harness-setting, ci, github-actions, frozen-file, managed-block, workflow-split]
related:
  - lint-staged-gitignore-silent-revert.md
  - https://github.com/coseo12/astro-simulator/pull/163
---

## 요약

harness-setting 의 frozen 파일(단일 전체 덮어쓰기 대상)에 프로젝트 고유 로직을 얹어야 할 때, 동일 파일에 섞지 말고 **별도 파일로 책임 경계를 분리**하고 `.harness/manifest.json` 추적에서 제외한다. 업스트림 업데이트 시 구조적으로 충돌이 생기지 않는다.

## 본문

### 문제

harness-setting v2.3.0 → v2.4.0 업데이트에서 `.github/workflows/ci.yml` 이 frozen 파일로 분류되어 전체 덮어쓰기 대상이었다. 업스트림 v2.4.0 ci.yml 은 제네릭 스캐폴딩(Rust/wasm 스텝 제거)으로 회귀했는데, 프로젝트(astro-simulator P3 Barnes-Hut Rust+WASM)가 추가한 다음 스텝이 덮어쓰기로 삭제될 위기였다:

- Rust 툴체인 설정 (`dtolnay/rust-toolchain@stable`, toolchain 1.94.1, wasm32 target)
- wasm-pack 0.14.0 설치
- `cargo test --release` (physics-wasm, 1000년 심플렉틱 적분)
- `verify:test-coverage` 가드

매니페스트는 이전 템플릿 해시 기준이라 "modified-pristine"(사용자 미수정)으로 **오분류**되어 `--apply-all-safe` 에 포함되어 버렸다.

### 해결 패턴 — 워크플로 파일 분리

```
.github/workflows/
├── ci.yml              # harness frozen 유지 (제네릭 스캐폴딩, 업스트림 관리)
└── ci-physics-wasm.yml # 프로젝트 로컬 (Rust/wasm-pack/verify:test-coverage)
```

1. `ci.yml` 을 업스트림 최신으로 되돌린다 (harness 관리에 맡김)
2. 프로젝트 고유 스텝만 담은 `ci-<project-slug>.yml` 을 신규 생성
3. 두 파일이 같은 트리거(`pull_request` → develop/main) 사용 → GitHub Actions 가 병렬 실행
4. `.harness/manifest.json` 의 `files` 객체에서 로컬 파일 엔트리를 **직접 삭제** → harness 가 추적하지 않음
5. 이후 `npx harness update --bootstrap` 실행 시에도 로컬 파일이 재추가되지 않도록 삭제 유지

### 트레이드오프

- ✅ 업스트림 업데이트 시 로컬 CI 로직이 구조적으로 건드려지지 않음
- ✅ 어떤 스텝이 harness 관리이고 어떤 스텝이 프로젝트 고유인지 책임 경계가 파일로 명확
- ⚠️ PR 체크가 2개로 분리 → 브랜치 보호 룰에 required check 2개 등록 필요
- ⚠️ manifest 수동 편집 단계 필요 (harness CLI 에 `.harnessignore` 같은 메커니즘 부재)

### 대안 비교

1. **(선택) 파일 분리** — 즉시 적용, 구조적 차단
2. managed-block 센티널 확장 — harness CLI 가 YAML 에서도 `# harness:managed:start/end` 블록을 파싱하도록 업스트림 PR 필요 (현재 CLAUDE.md 는 지원, YAML/기타 미지원)
3. `.harnessignore` 로컬 오버라이드 — 현재 `lib/categorize.js` 미지원

### 후속

harness-setting 에 워크플로/YAML 파일용 managed-block 센티널 지원 제안 이슈 필요.
