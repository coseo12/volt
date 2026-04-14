---
title: Rust + wasm-pack vs AssemblyScript — 수치 핫루프 언어 선택
type: decision
source_repo: coseo12/astro-simulator
source_issue: 5
captured_at: 2026-04-14
status: refined
tags: [wasm, rust, assemblyscript, performance]
related: []
---

## 배경/상황

P2 Newton N-body 적분기를 WASM으로 구현해야 했다. O(N²) 중력 합 + Velocity-Verlet 스텝의 핫루프가 프레임당 수만~수십만 번 실행되어 성능이 중요. 언어 후보: Rust, AssemblyScript, C++(Emscripten), TS.

## 내용

비교 축:
- **Rust**: 성능 최상, `wasm-pack` 성숙, `nalgebra` 등 수치 크레이트, 메모리 안전, 러닝커브 있음.
- **AssemblyScript**: TS 문법 그대로라 기존 pnpm 워크스페이스에 자연 통합. 빌드 단순. 생태계 얕고 수치 최적화 수동. 실성능 Rust 대비 70~80%.
- **C++ (Emscripten)**: 기존 N-body 레퍼런스 포팅 용이하지만 툴체인 무겁고 메모리 안전 취약.
- **TS without WASM**: 의존성 0, 빠른 시작. N=1000+ 60fps 달성 불가.

선정: **Rust + wasm-pack**.
- 수치 핫루프 성능 차이 체감. 측정: N=1000 1step native 2.7ms (WASM 1.5~2× 느림 감안해도 프레임 예산 내).
- P3 WebGPU compute(Barnes-Hut 등)로 확장 시 Rust 자산 재활용 가능.
- `rust-toolchain.toml`로 버전 pin → 재현성 확보.
- CI: `cargo install wasm-pack --locked` + `dtolnay/rust-toolchain` 조합으로 안정.

조건부 exports로 Node/bundler 타깃 모두 지원:
```json
"exports": {
  ".": {
    "node": { "default": "./pkg/physics_wasm.js" },
    "default": "./pkg-bundler/physics_wasm.js"
  }
}
```

## 교훈 / 다음에 적용할 점

- **수치 핫루프(O(N²)/쉐이더성 계산)가 있는 웹 프로젝트는 Rust + wasm-pack이 기본값.**
- AssemblyScript는 "빠른 프로토타이핑 + 가벼운 WASM" 영역에서 여전히 유효하지만, P2-D처럼 성능 목표가 명확하면 Rust 직행이 유리.
- `rust-toolchain.toml` + `cargo install --version --locked`로 CI/로컬 버전 고정은 의무.
- Node vs bundler 타깃은 조건부 exports로 분리 (테스트 Node, 앱 bundler). skill/가이드에 템플릿 포함 가치.
