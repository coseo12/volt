---
title: 상태 기록 원자성 — 도중/사후/안내 3단 다계층 방어 패턴
type: knowledge
source_repo: coseo12/harness-setting
source_issue: 28
captured_at: 2026-04-18
status: refined
tags: [atomicity, state-integrity, multi-layer-defense, manifest, post-apply-verification, self-healing, user-visibility]
related:
  - ./cross-validate-accept-vs-defer.md
  - ./manifest-state-recovery-pattern.md
  - ./lint-staged-gitignore-silent-revert.md
  - ../meta/patterns/cross-validate-after-policy-freeze.md
  - ./gitflow-drift-4tier-classifier.md
  - ./fourth-automation-mapping-layer.md
  - ./monorepo-core-dist-stale.md
---

## 요약

원자성이 요구되는 상태 기록 시스템(매니페스트, 캐시, 인덱스) 에서 단일 방어선은 **타이밍 blind spot** 때문에 부족하다. **3계층 직교 방어** 로 설계하라: (1) 연산 도중 외부 간섭 감지, (2) 연산 이후 상태 드리프트 자가 복구, (3) 복구 가능 신호를 사용자에게 명시 노출. harness-setting 의 매니페스트 원자성 완성 과정(v2.8.0 → v2.9.0 → v2.10.0) 이 실증.

## 본문

#### 배경

volt [#27](https://github.com/coseo12/volt/issues/27) 에서 관찰된 "매니페스트 최신 ≠ 파일 적용 완료" 교착 상태를 코드 레벨로 해소하는 과정에서 **단일 해결책의 타이밍 blind spot** 이 드러났고, 3계층 방어로 자연스럽게 귀결됐다.

#### 3계층 패턴

**계층 1 — 도중 방어 (post-apply 검증 게이트, harness v2.8.0)**
- 상태 갱신 연산 직후 기대 해시와 실측 해시를 비교
- 불일치 시 해당 엔트리의 상태 기록을 갱신하지 않음 (이전 값 유지)
- 커버: 연산 프로세스 종료 전 외부 간섭 (즉시 실행되는 워커, 병렬 편집 등)
- blind spot: 연산 종료 **이후** 에 발생하는 간섭(git commit 시점 lint-staged 훅) 은 못 잡음

**계층 2 — 사후 복구 (previousSha256 자가 복구, harness v2.9.0)**
- 각 엔트리에 "직전 상태 해시" 를 optional 필드로 기록
- 다음 연산 시 현재 파일 해시가 previousSha256 과 일치하면 "외부 롤백" 확정 → 재적용 허용
- 커버: 계층 1 이 못 잡는 사후 간섭을 **다음 연산에서** 자가 해소
- blind spot: 자가 복구가 가능하다는 사실을 사용자가 모르면 수동 개입(매니페스트 git checkout 복구) 하게 됨

**계층 3 — 사용자 안내 (doctor 분류, harness v2.10.0)**
- 상태 점검 명령(doctor) 이 "복구 가능" 신호를 별도 카테고리로 노출
- "외부 롤백 의심 N건 — `--apply-all-safe` 로 자가 복구 가능" 메시지
- 커버: 계층 2 의 복구 경로가 사용자에게 가시적이 되어 즉시 활용됨

#### 일반화 — 다른 시스템 적용

| 시스템 | 계층 1 | 계층 2 | 계층 3 |
|--------|-------|-------|-------|
| 파일 시스템 인덱스 | fsync 후 checksum 검증 | journal/previous-state | `fsck` 가 복구 가능 항목 분리 |
| DB 마이그레이션 | post-migrate assertion | shadow table / down-migration | `rake db:status` 가 드리프트 노출 |
| 빌드 캐시 | 빌드 후 산출물 해시 검증 | 이전 빌드 해시 보존 | `--diagnose` 가 캐시 오염 분리 |
| git 서브모듈 | commit 후 submodule 상태 체크 | 이전 submodule SHA 기록 | `git status` 가 drift 명시 |

#### 적용 조건

- **계층 1 만으로 충분**: 연산 프로세스가 짧고 외부 간섭 창이 연산 내부로만 제한됨
- **계층 1+2**: 연산 종료 이후에도 상태가 변조될 가능성 있음 (pre-commit 훅, 비동기 워커)
- **3계층 모두**: 자가 복구가 사용자 action 을 요구하고, 자동 실행되지 않을 때 (안내 없으면 복구 기능이 사장됨)

#### 박제 근거

- harness [#89](https://github.com/coseo12/harness-setting/issues/89) (v2.8.0 계층 1)
- harness [#92](https://github.com/coseo12/harness-setting/issues/92) Phase 1 (v2.9.0 계층 2) + Phase 2 (v2.10.0 계층 3)
- Gemini 교차검증에서 제안된 previousSha256 이 계층 2 로 이어졌고, 이를 공개하는 안내가 계층 3

## 관련 노트/링크

- volt [#27](https://github.com/coseo12/volt/issues/27) — 원자성 교착 상태 원본 관찰 (계층 1/2 구현 동기)
- volt [#13](https://github.com/coseo12/volt/issues/13) — lint-staged silent partial commit (계층 2 의 방어 대상 타이밍)
- volt [#23](https://github.com/coseo12/volt/issues/23) — 교차검증 루틴 (계층 2 설계는 Gemini 교차검증에서 도출)
- harness releases: [v2.8.0](https://github.com/coseo12/harness-setting/releases/tag/v2.8.0), [v2.9.0](https://github.com/coseo12/harness-setting/releases/tag/v2.9.0), [v2.10.0](https://github.com/coseo12/harness-setting/releases/tag/v2.10.0)
