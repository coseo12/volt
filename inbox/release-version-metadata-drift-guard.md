---
title: 릴리스 metadata 이중 업데이트 드리프트 — CHANGELOG vs package.json::version 암묵 관례의 구조적 검증 승격
type: knowledge
source_repo: coseo12/harness-setting
source_issue: 56
captured_at: 2026-04-20
status: inbox
tags: [release-pipeline, metadata-drift, ci-guard, implicit-convention, recovery-pattern, structural-enforcement, silent-failure]
related:
  - ../notes/lint-staged-gitignore-silent-revert.md
  - ../notes/manifest-state-recovery-pattern.md
---

## 요약

릴리스 파이프라인에서 **여러 파일을 함께 bump 해야 하는 암묵 관례** (본 사례: `CHANGELOG.md` + `package.json::version`) 가 과거 N 회 일관되게 지켜지다가 특정 세션에서 일관되게 누락될 수 있다. 누락은 "조용한 실패" — 태그·release·CI 모두 green 이지만 다운스트림이 metadata 를 구 버전으로 인식. 복구는 PATCH 1개로 가능하지만 재발 방지를 위해 **암묵 관례를 구조적 검증 (CI 가드 스크립트) 으로 승격** 하는 것이 본질적 해결. harness-setting v2.28.1 에서 실측.

## 본문

#### 관찰 사례

- **과거 관례**: v2.22.1 ~ v2.25.0 chore release 커밋 4건 전부 `package.json::version` bump 포함 (`git show <commit> -- package.json` 로 확인 — `-"version": "2.24.0" / +"version": "2.25.0"` 류)
- **일관된 누락**: v2.26.0 / v2.27.0 / v2.28.0 chore release 3건 모두 `package.json::version` bump 생략. `CHANGELOG.md` 엔트리만 추가
- **조용한 실패 신호 (모두 green 이었던 지점)**:
  - `git tag` + `gh release create` 모두 성공 (태그·release 메타데이터는 package.json 과 독립)
  - CI `detect-and-test` 통과 (package.json version 검증 없었음)
  - 로컬 `npm test` 통과 (version 필드와 무관)
  - 다운스트림 `harness update` 가 파일 해시 기반으로는 최신 감지 → 콘텐츠 drift 는 0
- **발견 경로**: 사용자가 "다른 저장소에서 업데이트 시 버전 불일치" 보고 → 실측으로 `package.json::version = 2.25.0` 확인 → 과거 릴리스 diff 와 대조하여 관례 누락 특정

#### 근본 원인 — "암묵 관례" 의 3가지 취약성

1. **절차 문서화 부재** — CLAUDE.md `### 릴리스` 섹션이 tag + release 만 명시, `package.json::version` bump 는 미명시. 과거 N 회 성공은 "숙련된 기억" 에만 의존
2. **검증 없는 관례** — 관례를 깼을 때 알려주는 신호가 없음. CI / release 도구 / 태그 생성 모두 version 필드와 독립
3. **세션 피로 / 속도** — 연속 릴리스 (1 시간 3 릴리스) 에서 절차 체크리스트가 줄어들며 암묵 관례가 먼저 탈락

#### 수정 및 회귀 가드 (v2.28.1)

**즉시 복구**:
- `package.json::version: 2.25.0 → 2.28.1`
- CHANGELOG `## [2.28.1]` PATCH 엔트리 (Behavior Changes: None, metadata only)

**회귀 가드 구조적 승격**:

```bash
#!/usr/bin/env bash
# scripts/verify-release-version-bump.sh (요지)
# CHANGELOG 의 첫 `## [X.Y.Z]` 엔트리 추출 → package.json::version 과 비교
# 불일치 시 exit 1 + 수정 안내 (A) / (B) 양방향 제시

pkg_version=$(jq -r '.version' package.json)
changelog_version=$(grep -oE '^## \[[0-9]+\.[0-9]+\.[0-9]+\]' CHANGELOG.md | head -1 | sed -E 's/## \[(.+)\]/\1/')
[ "$pkg_version" = "$changelog_version" ] || { echo "drift"; exit 1; }
```

- `.github/workflows/ci.yml` `detect-and-test` 에 통합 (agent SSoT 가드 #145 와 동일 패턴)
- CLAUDE.md `### 릴리스` 에 `package.json::version` bump 필수 규약 + 가드 경로 박제

#### 일반화 패턴 — "암묵 관례의 구조적 검증 승격" 4단계

| 단계 | 질문 | 행동 |
|---|---|---|
| 1. 관례 발견 | "이 관례가 과거 N 회 지켜져 왔는가?" | git log / diff 로 실측. 5회 이상이면 관례 확인 |
| 2. 누락 탐지 | "누락 시 어떤 신호가 있는가?" | CI·테스트·태그·release 중 **자동 실패 신호가 있는지**. 없으면 조용한 실패 고위험군 |
| 3. 검증 승격 | "1회 실측 가능한 규칙으로 표현 가능한가?" | 두 위치 값 비교 / 패턴 매칭 / 스키마 validate. 표현 가능하면 스크립트 작성 |
| 4. 파이프라인 통합 | "어느 시점에 실행해야 최대 효과인가?" | CI PR 차단 (가장 엄격) / pre-commit 훅 / local 권장 명령 순으로 체인 구성 |

#### 다른 적용 시나리오

- **모노레포 workspace version 동기화** — 루트 + 각 패키지 `package.json::version` 일치 검증
- **Docker 이미지 tag ↔ `VERSION` 파일 일치** — Dockerfile `LABEL version=` 과 VERSION 파일
- **DB migration checksum** — `schema_migrations` 테이블 row 과 `db/migrate/*.sql` 파일 checksum 일치
- **문서 버전 ↔ 코드 버전** — README `## Version X.Y` 와 패키지 version
- **API version header ↔ schema 버전** — 응답 header 과 OpenAPI spec 버전
- **lock file ↔ manifest file** — `Cargo.lock` vs `Cargo.toml` 의 종속성 버전

공통 조건 (본 패턴 적용 가능 신호):
- 두 위치가 **동일 값** 을 가져야 하지만 편집 시점이 다름
- 편집 주체가 **다른 타이밍** 에 각각 수정 (한 명의 작업자가 두 번 편집하는 경우에도 누락 가능)
- 불일치 시 **즉시 실패가 아닌 다운스트림 drift** 로 나타남 (조용한 실패)

#### 연쇄 관계

- volt [#13](https://github.com/coseo12/volt/issues/13) "커밋 성공 ≠ 의도한 변경 커밋됨" — lint-staged silent partial commit 의 릴리스 파이프라인 버전
- volt [#27](https://github.com/coseo12/volt/issues/27) "매니페스트 최신 ≠ 파일 적용 완료" — 파일 적용과 매니페스트 기록이 원자적 트랜잭션이 아닌 문제의 일반화
- harness-setting v2.23.0 `verify-agent-ssot.sh` — "SSoT 5개 파일 × N 필드 일치 검증" 의 동일 구조 패턴 선례

## 교훈 / 다음에 적용할 점

1. **"이번이 예외" 편향 경계** — 과거 N 회 성공했다고 다음 번 안 되는 것이 아님. 세션 피로·속도 압박에서 관례가 먼저 탈락
2. **자동화 수준의 sweet spot** — 완전 자동화 (version bump 스크립트화) 가 아니라 **수동 작업 + 사후 검증** 이 실용적. 작업자가 의식적으로 bump 한 후 스크립트가 실측 검증하는 이원화
3. **복구 PATCH 자체의 "None Behavior Change" 선언** — metadata only 복구도 PATCH 릴리스로 명시. `Behavior Changes: None — metadata/가드만` 섹션 필수. 다운스트림이 "어? X.Y.Z → X.Y.(Z+1) 왜?" 오해하지 않도록 설명
4. **세션 연속 릴리스의 체크리스트 밀도 관리** — 1 시간 내 3 릴리스 같은 속도에서는 체크리스트를 간소화하지 말고 오히려 **CI 자동 차단으로 옮겨 체크리스트 밀도와 무관하게 검증**

## 관련 노트/링크

- harness PR [#173](https://github.com/coseo12/harness-setting/pull/173) (복구 + CI 가드 도입)
- harness PR [#174](https://github.com/coseo12/harness-setting/pull/174) (v2.28.1 release)
- harness release [v2.28.1](https://github.com/coseo12/harness-setting/releases/tag/v2.28.1)
- 선행 volt [#13](https://github.com/coseo12/volt/issues/13), volt [#27](https://github.com/coseo12/volt/issues/27)
- harness v2.23.0 `verify-agent-ssot.sh` (동일 구조 패턴 선례)
