---
title: 매니페스트 기반 패키지 관리자 재시도 — manifest/상태 파일 복구 패턴
type: knowledge
source_repo: coseo12/astro-simulator
source_issue: 27
captured_at: 2026-04-18
status: refined
tags: [package-manager, manifest, state-recovery, idempotency, harness-setting, partial-failure]
related:
  - ./lint-staged-gitignore-silent-revert.md
  - ./state-atomicity-multi-layer-defense.md
  - ./harness-downstream-prettier-drift.md
---

## 요약

매니페스트 기반 패키지 관리자(harness-setting, Nix, brew 등)에서 **부분 실패 후 재시도가 스킵되는 함정**. 파일 적용과 매니페스트 해시 기록이 별도 단계이므로, 파일 적용이 일부 롤백되어도 매니페스트는 최신 버전을 기록해 다음 `--apply-all-safe`가 "동일 상태"로 판단하고 재-apply를 스킵한다. 해결: 매니페스트를 이전 커밋에서 `git checkout`으로 복구한 뒤 재-apply.

## 본문

### 증상

1. `harness update --apply-all-safe`로 27개 파일 업데이트 시도
2. lint-staged 부분 실패로 22개 파일이 working tree 상태로 롤백 (volt [#13](https://github.com/coseo12/volt/issues/13) 참조)
3. 그러나 `.harness/manifest.json`은 이미 v2.7.0 해시로 갱신되어 커밋에 포함됨
4. 재-apply 시도 (`harness update --check`) → "사용자 변경 보존 (패키지 미변경)"으로 22개 파일을 **사용자 임의 수정**으로 간주
5. `--apply-all-safe`는 해당 카테고리를 건너뛰므로 복구 불가능한 교착 상태

### 원인

- 매니페스트의 해시는 "패키지 상위가 제공하는 기대 상태"를 의미
- 패키지 적용 시 매니페스트 갱신이 파일 적용과 **원자적(atomic) 트랜잭션이 아님**
- 파일 적용이 실패해도 매니페스트는 성공으로 기록될 수 있음
- 재시도 시 매니페스트가 "이미 최신"이라는 전제로 진입하므로 차이 감지 실패

### 일반화

동일 패턴은 다음 시스템에서 나타날 수 있다:

- **Nix**: `/nix/var/nix/profiles/*` 링크 업데이트와 스토어 파일 빌드 분리
- **brew**: Cellar 설치 + Formula 버전 기록 분리
- **dpkg/apt**: `/var/lib/dpkg/status` vs `/var/cache/apt/archives/*` 분리
- **npm package-lock.json**: 해시 기록 vs `node_modules/` 실체 파일

### 해결

1. **즉시 복구**:

   ```bash
   git checkout <이전-머지-커밋> -- .harness/manifest.json
   npx github:coseo12/harness-setting update --apply-all-safe
   # 이제 22개 파일이 다시 "pristine"으로 감지되어 재적용됨
   ```

2. **근본 대응** (패키지 관리자 설계):
   - 파일 적용 완료 후 매니페스트 갱신 (커밋 순서 역전)
   - 또는 파일 적용 실패 시 매니페스트도 롤백 (원자적 트랜잭션)
   - 사용자 관점: 재시도 가능성 보장이 idempotency보다 우선

3. **예방 (사용자 루틴)**:
   - 패키지 업데이트 커밋 시 매니페스트와 파일을 동일 커밋에 묶기
   - 부분 실패 감지 시 전체 revert + 재시도가 부분 보수보다 안전

## 관련 노트/링크

- volt [#13](https://github.com/coseo12/volt/issues/13) — lint-staged silent partial commit (선행 문제)
- astro-simulator [PR #202](https://github.com/coseo12/astro-simulator/pull/202) — 본 복구 패턴 적용 PR
- harness-setting 내부: `.harness/manifest.json` 해시 갱신 시점 설계 재검토 필요
- Nix 공식 문서: [rollback & generations](https://nixos.org/manual/nix/stable/package-management/profiles.html)
