---
title: harness-managed 파일과 다운스트림 prettier/lint-staged 통합 경계 — drift 반복 패턴
type: report
source_repo: coseo12/astro-simulator
source_issue: 35
captured_at: 2026-04-18
status: inbox
tags: [harness, prettier, lint-staged, integration-boundary, drift, update-flow]
related:
  - ../notes/lint-staged-gitignore-silent-revert.md
  - ../notes/manifest-state-recovery-pattern.md
---

## 배경/상황

astro-simulator 에서 `harness update --apply-all-safe` 로 v2.7.0 → v2.11.0 적용 중 관찰. lint-staged 의 `prettier --write` 가 upstream 의 스타일 변경(single→double quote, 섹션 헤더 뒤 빈 줄 삭제, 테이블 공백 정렬 등)을 로컬 prettier 컨벤션으로 되돌려, 35개 파일이 실질 콘텐츠 차이 없이 포맷만 다른 상태로 롤백되었다. 매니페스트는 upstream packageSha 로 기록되지만 로컬 파일은 prettier 포맷 → `harness update --check` 가 지속적으로 "안전 업데이트 35+개"를 노이즈로 표시한다.

harness-setting 자체의 버그는 아니다(자기 패키지 포맷은 일관됨). 다운스트림 프로젝트의 prettier 설정도 각자 합리적이다. 문제는 두 시스템의 **통합 경계에서 합의 부재** — harness-managed 파일이 프로젝트 lint-staged 파이프라인을 거치면서 재포맷되는 것.

## 내용

**재현 경로 (PR #228)**

1. `feature/harness-v2.11.0` 브랜치 생성
2. `harness update --apply-all-safe` 로 40+ 파일 덮어쓰기 (pristine + added + managed-block)
3. `harness update --bootstrap` 후 추가로 35 파일 drift 감지 → 2차 apply
4. `git commit` → lint-staged `prettier --write` 가 markdown 파일들의 quote·공백·빈 줄을 로컬 스타일로 재포맷
5. 실질 콘텐츠 변경이 있던 4 파일만 최종 반영 (CLAUDE.md, pm.md, cross-validate SKILL.md, 매니페스트)
6. `--check` 재실행 시 35 파일이 다시 drift 로 감지 → 다음 업데이트에서도 반복 예상

**관찰된 항목**

- `.prettierrc.json` : `singleQuote: true`, `printWidth: 100`
- lint-staged 패턴 : `*.{json,md,yaml,yml,css}` → `prettier --write`
- upstream 포맷    : `singleQuote: false`, 섹션 헤더 뒤 빈 줄 없음, 테이블 공백 정렬 없음
- `.prettierignore` : `.harness/` 만 포함, `.claude/` / `.github/` 미포함

**영향**

- `harness update --check` 신호 대비 노이즈 비율 악화 → 실질 upstream 변경을 놓칠 위험
- `--bootstrap` 재박제로 일시 무마는 가능하나 다음 업데이트마다 재발
- 커밋 메시지가 "39 파일 동기화" 를 명시했어도 실제로는 4 파일만 반영되는 현상 (volt #13 "커밋 성공 ≠ 의도한 변경 커밋됨" 의 새로운 실례)

## 교훈 / 다음에 적용할 점

**다운스트림 프로젝트 측 대응 (astro-simulator issue #229 로 분리)**

후보 A: `.prettierignore` 에 harness-managed 경로 추가 (`.claude/`, `.github/ISSUE_TEMPLATE/`, `.github/PULL_REQUEST_TEMPLATE.md`, harness-관리 `docs/*.md` 목록) — 가장 자연스러운 선택
후보 B: 프로젝트 `.prettierrc.json` 을 upstream 포맷 호환으로 조정 — 기존 코드베이스 diff 과다, 비현실적
후보 C: 매니페스트에서 관리 파일 목록 자동 추출 → `.prettierignore` 스니펫 생성 스크립트 — 유지비 있음

**harness-setting 측 보강 제안 (권고성)**

- **README/docs 가이드 추가**: "다운스트림 프로젝트는 `.prettierignore` 에 harness-managed 경로 추가 권장 (drift 방지)" — 최소 비용, 즉각 효과
- **`harness doctor` 진단 보강**: 프로젝트 루트의 `.prettierrc*` / `package.json` lint-staged 설정 감지 시, harness-managed 경로가 `.prettierignore` 에서 제외되지 않았으면 경고 출력
  - 예: `⚠  prettier lint-staged 감지됨 — .prettierignore 에 .claude/ 등 harness 경로 추가 권장 (drift 방지)`
- **설치 시점 가이드**: `harness update --bootstrap` 실행 후 "다음 단계" 안내에 lint-staged/prettier 통합 힌트 포함
- **자동화 옵션 (더 큰 스코프)**: `harness gen-prettierignore` 서브커맨드로 매니페스트에서 관리 파일 목록을 뽑아 `.prettierignore` 스니펫 생성

**대응 우선순위 (harness 쪽)**

1. 최소 가치: README/docs 가이드 (낮은 비용 / 즉각 효과)
2. 중간 가치: `doctor` 경고 (재발 예방 자동화)
3. 높은 가치: 자동 생성 스크립트 (유지비 존재)

**일반화된 교훈**

- 매니페스트 기반 패키지가 다운스트림의 파일 처리 파이프라인(lint-staged, pre-commit hooks, formatter)과 만나는 경계에서 drift 가 발생한다
- `harness update` 류 명령의 "적용 완료"는 `git commit` 까지 검증이 필요 — 파이프라인 개입으로 실질 커밋 내용이 달라질 수 있음
- 볼트 [#13](https://github.com/coseo12/volt/issues/13) (.gitignore 로 인한 유실) 과 본 건(prettier 재포맷)은 다른 기전이지만 같은 원칙: "staging 성공 ≠ 커밋 확정 내용"

## 관련 노트/링크

- astro-simulator PR #228 — 본 관찰의 트리거
- astro-simulator issue #229 — 다운스트림 해결 이슈
- volt [#13](https://github.com/coseo12/volt/issues/13) — "커밋 성공 ≠ 의도한 변경 커밋됨" 원칙의 새로운 실례
- volt [#27](https://github.com/coseo12/volt/issues/27) — 매니페스트 부분 실패 교착 (관련 주제 / 기전 다름)
- harness-setting v2.11.0 CLAUDE.md "매니페스트 최신 ≠ 파일 적용 완료" 섹션
