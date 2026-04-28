---
title: sub-agent 프로세스 리크 — run_in_background 좀비 누적 (#24 연장선)
type: pattern
source_repo: coseo12/astro-simulator
source_issue: 52
captured_at: 2026-04-20
status: refined
tags: [sub-agent, process-management, cargo, background-process, workflow-pattern, harness-rag]
related:
  - ./sub-agent-finalization-miss-pattern.md
  - ./agent-browser-chrome-zombie-pattern.md
  - ../../notes/long-test-ignore-fast-loop-template.md
  - ../../notes/reviewer-qa-gate-skip-headless-self-pass.md
---

## 배경/상황

**volt #24 (sub-agent 마무리 보고 누락 반복 패턴) 의 프로세스 레벨 확장.** #24 는 sub-agent 가 코멘트·라벨·PR 생성 등 GitHub 외부 가시성 박제를 누락하는 패턴을 다룬다. 본 사건은 같은 "sub-agent 반환 전 마무리 불완전" 원칙이 **`run_in_background=true` 로 띄운 로컬 프로세스 정리** 레이어로 확장된 관찰.

P9 (목성계) PR-1 (astro-simulator#258) 진행 중, dev(초기 구현) → reviewer → dev(재작업) → qa 4 단계 sub-agent 호출이 이어지면서 각자 `cargo test --lib` 를 독립적으로 백그라운드 실행. 어느 sub-agent 도 반환 직전 자신이 띄운 프로세스의 완주/정리를 확인하지 않음. 메인 오케스트레이터도 복귀 후 프로세스 상태 독립 확인 루틴 부재.

## 내용

#### 관찰된 좀비 누적

| PID | 시작 | 경과 | CPU | 책임 sub-agent |
|---|---|---|---|---|
| 65189 | 13:00 | 176분 | 94.5% | dev (PR-1 초기 검증) |
| 71420 | 13:10 | 167분 | 93.3% | dev (재작업 중 재실행) |
| 78434 | 13:28 | 154분 | 268.8% | reviewer |
| 85117 | 13:45 | 125분 | 387.8% | qa |

4개 `physics_wasm-<hash>` 테스트 바이너리가 **동일 target 디렉토리 락 경쟁**. 각 프로세스는 CPU 수백% 점유하나 디스크 I/O 교착으로 어느 것도 완주 못 함. 정상 소요 4~5분 대비 30~176분 경과에도 진행 정체.

좀비 3개 kill 후 단일 프로세스로 복귀시키자 ~8분 내 완주 (CPU 397% → 실제 코어 활용 정상화).

#### 근본 원인

1. **sub-agent 마무리 책임 불명확**: sub-agent 는 "배경 작업 시작"과 "완주 확인" 사이에 반환할 수 있으나, 이 인계가 메인 오케스트레이터에게 명시적으로 전달되지 않음
2. **메인 복귀 시 독립 확인 부재**: sub-agent 보고서의 "cargo 진행 중" 기재만 신뢰, `ps auxww` 독립 확인 없음
3. **동일 target 경쟁의 침묵 실패**: Rust 빌드 락이 에러 대신 silent wait → 외부에서 "진행 중" vs "교착" 구분 불가
4. **장기 테스트 혼재**: `cargo test --lib` 에 4~5분 빠른 테스트와 60초+ long-running 테스트가 섞여있어 timeout 설정이 어려움

#### 해소 방안 (astro-simulator 로컬 적용)

- **단기** (astro-simulator#259): CLAUDE.md §프로젝트 고유 보강 교훈 에 본 교훈 박제 — 메인 루틴 / sub-agent 체크리스트 규범
- **중기**: 장기 적분 테스트 (`mercury/yoshida_*_perihelion_*`, `earth/venus_perihelion_eih_*`) 에 `#[ignore]` 어트리뷰트 + CI 전용 `--include-ignored` 경로 (P9 PR-2 에서 도입 예정)
- **장기** (본 이슈): harness upstream agents 규약 반영

## 교훈 / 다음에 적용할 점

#### harness agents 규약 (.claude/agents/*.md) 반영 제안

1. **sub-agent 마무리 체크리스트 JSON 필드 확장**:
   ```json
   {
     "spawned_bg_pids": [85117],
     "bg_process_handoff": "메인 정리 책임" | "sub-agent 완주 확인 완료"
   }
   ```
   `run_in_background=true` 로 시작한 PID 를 명시적으로 반환하여 메인이 정리 책임 인지

2. **메인 오케스트레이터 루틴 (CLAUDE.md 또는 주 agents 규약)**:
   ```bash
   # sub-agent 복귀 직후 의무
   ps auxww | grep -E "cargo|next dev|physics_wasm-|vitest|jest" | grep -v grep
   ```
   현재 세션 이전 시작 시각 프로세스 발견 시 kill 또는 완주 확인

3. **sub-agent 자체 규범**:
   - 반환 직전 `pgrep -f <자신이 띄운 프로세스 패턴>` 실행
   - 완주 확인 못 하면 보고서에 "배경 작업 인계" 플래그 + 메인 오케스트레이터 정리 절차 참조

4. **타겟 디렉토리 경쟁 감지 루틴**:
   - Rust 의 경우 `target/` 디렉토리 락 파일 (`target/debug/.rustc_info.json` 접근 lock) 다수 검출 시 경고
   - npm/pnpm 의 경우 `node_modules/.cache/` 동시 쓰기 감지

5. **장기 테스트 분리 규범** (언어 중립):
   - Rust: `#[ignore]` + CI `--include-ignored`
   - Jest/Vitest: `describe.skip` 또는 `test.todo` + `--runInBand`
   - 일상 개발 경로는 항상 5분 내 완주하도록 재설계

#### volt #24 와의 관계

- #24: GitHub 가시성 레이어 (코멘트/라벨/PR)
- 본 이슈: 로컬 프로세스 레이어
- **공통 원리**: sub-agent 의 "완료" 와 시스템 관점 "완료" 불일치. 명시적 인계 계약이 없으면 누락 반복
- harness 차원에서 두 레이어를 **단일 마무리 체크리스트** 로 통합 관리 권고

#### 재발 감지 신호

- sub-agent 여럿 호출한 세션에서 `cargo test --lib` / `pnpm test` 가 이전 대비 5배+ 소요
- 동일 이름 테스트 바이너리가 `ps` 에서 2개 이상 동시 관찰
- `target/debug/` 크기가 기대치 대비 비정상 증가 (각 프로세스가 partial 빌드 아티팩트 남김)

## 참고

- 선행: [volt #24](https://github.com/coseo12/volt/issues/24) — sub-agent 마무리 보고 누락 패턴 (4회 관찰)
- 관찰 PR: [astro-simulator#258](https://github.com/coseo12/astro-simulator/pull/258) — P9 PR-1 (머지 완료)
- 단기 가드 PR: [astro-simulator#259](https://github.com/coseo12/astro-simulator/pull/259) — CLAUDE.md 로컬 박제
