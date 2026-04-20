---
title: 범용 CI 다언어 지원 설계 4원칙 — lock 우선순위 배타 조건 + 공급망 보안 + 명시 의존성 + drift 각주
type: knowledge
source_repo: coseo12/harness-setting
source_issue: 57
captured_at: 2026-04-20
status: inbox
tags: [ci, multi-language, package-manager, supply-chain-security, yarn-berry, python-package-managers, rust-cargo, go, pattern-extraction, echo-only-trap, review-actionability]
related:
  - ../notes/ci-green-no-test-execution-trap.md
  - ../notes/comment-contract-implementation-drift.md
  - ../notes/external-tool-claim-empirical-verify.md
  - ../notes/long-test-ignore-fast-loop-template.md
  - ../notes/cross-validate-principle-declaration-bias.md
---

## 요약

범용 CI 템플릿에서 다언어 지원을 설계할 때 (1) 실 실행 vs echo-only 트랩 (volt [#48](https://github.com/coseo12/volt/issues/48) 재현 방지), (2) 같은 언어 내 여러 패키지 관리자의 **lock 파일 우선순위 기반 배타 조건 분기**, (3) 설치 스크립트의 공급망 보안 고려 (`curl | sh` 금지 → 공식 action), (4) 도구 체인의 **명시적 의존성 설치** (pytest 등 암묵 전제 금지) 4가지 관점이 서로 독립적으로 검증되어야 한다. harness-setting v2.29.0 에서 4개 축 (Python + Go + Rust + yarn berry/classic) 을 동시 적용한 실측 사례. reviewer / Gemini cross-validate 가 각 관점을 분담 포착.

## 본문

#### 1. lock 파일 우선순위 기반 배타 조건 분기 (Node.js / Python 공통 패턴)

**원리**: 같은 언어에 여러 패키지 관리자가 공존 가능할 때 (Node: npm/pnpm/yarn/bun/deno / Python: pip/poetry/pipenv/uv) lock 파일 존재 여부로 단일 경로를 결정한다. CI 의 `if:` 조건을 **상위 lock 부재** 로 누적하여 배타성 보장.

```yaml
# Node.js 예시 — pnpm > yarn > npm lock > npm fallback
- name: pnpm ...
  if: hashFiles('pnpm-lock.yaml') != ''
- name: yarn ...
  if: hashFiles('yarn.lock') != '' && hashFiles('pnpm-lock.yaml') == ''
- name: npm ci
  if: hashFiles('package-lock.json') != '' && hashFiles('pnpm-lock.yaml') == '' && hashFiles('yarn.lock') == ''
- name: npm install (fallback)
  if: hashFiles('package.json') != '' && hashFiles('package-lock.json') == '' && hashFiles('pnpm-lock.yaml') == '' && hashFiles('yarn.lock') == ''
```

**주의**: `setup-node@v4` / `setup-python@v5` 의 `cache: 'npm'`/`'pnpm'`/`'yarn'`/`'pip'` 지시어는 해당 lock 파일 부재 시 **실패** (volt [#51](https://github.com/coseo12/volt/issues/51) 실측 확인). 단일 step 으로 통합하려는 유혹이 있으나 실측 반증 사례.

#### 2. yarn berry vs classic 분기 (`.yarnrc.yml` 유무)

yarn v1 (classic) 과 v2+ (berry) 는 install 명령이 다르다:
- classic: `yarn install --frozen-lockfile`
- berry: `yarn install --immutable` (v4+ 는 `--frozen-lockfile` 제거됨)

감지 규약: `.yarnrc.yml` 파일 존재 유무:
```yaml
- name: yarn install (berry)
  if: hashFiles('yarn.lock') != '' && hashFiles('.yarnrc.yml') != ''
  run: yarn install --immutable

- name: yarn install (classic)
  if: hashFiles('yarn.lock') != '' && hashFiles('.yarnrc.yml') == ''
  run: yarn install --frozen-lockfile
```

추가로 `corepack enable` 을 호출하면 `packageManager` 필드 우선 존중 (yarn 버전 자동 설정).

#### 3. Python 6 도구 분기 — lock 우선순위

```
uv.lock > poetry.lock > Pipfile.lock > requirements.txt > pyproject.toml (PEP 621) > setup.py (legacy)
```

**pytest 명시 설치 필수** — poetry/pipenv 경로에서 `poetry install` / `pipenv sync --dev` 가 dev-dependencies 에 pytest 를 **포함한다고 암묵 전제** 하면 `poetry run pytest` 가 command not found. 방어 패턴:

```bash
poetry run python -c "import pytest" 2>/dev/null || poetry run pip install pytest
poetry run pytest || [ $? = 5 ]
```

**pytest exit code 5** (수집 테스트 0건) 은 CI 실패가 아닌 **경고** 로 처리. 파이썬 테스트 미작성 프로젝트가 CI 를 막는 false positive 방지.

#### 4. `python-version-file` 의 함정

```yaml
uses: actions/setup-python@v5
with:
  python-version-file: 'pyproject.toml'   # ← [project].requires-python 필드 부재 시 실패
```

`pyproject.toml` 에 Python 버전 필드가 없는 프로젝트도 많다 (`[tool.poetry].python` 과 `[project].requires-python` 이 둘 다 없으면 실패). **안전한 기본값**: `python-version: '3.x'` fallback.

#### 5. 공급망 보안 — `curl | sh` 금지, 공식 action 우선

uv 설치 예시:

```yaml
# ❌ 보안 리스크 (MITM / 도메인 탈취 시 CI 임의 코드 실행)
run: curl -LsSf https://astral.sh/uv/install.sh | sh

# ✅ 공식 action (버전 고정 + 캐싱 + 공급망 공격 표면 축소)
uses: astral-sh/setup-uv@v3
with:
  enable-cache: true
```

Gemini cross-validate 가 이 지점을 첫 Approve 주의로 지적 → 수용. 차선안인 "checksum 검증" 보다 공식 action 이 운영 오버헤드 낮음.

#### 6. Rust: `--release` vs debug 구분

CI 일상 경로는 **debug 빌드 기본** (`cargo test --lib`). `--release` 는 빌드 시간이 수 배 더 길어 일상 회귀 탐지용으로 부적합. release 빌드 검증이 필요한 프로젝트는 별도 job 으로 분리 권장. 또한 장기 테스트 이원화 (volt [#54](https://github.com/coseo12/volt/issues/54)) 와 조합 가능.

#### 7. Bun / Deno — "감지 테이블 정의 / CI 미구현" 의도적 drift

범용 CI 템플릿에서는 수요가 확인되지 않은 생태계를 **의도적으로 미구현** 하되, 감지 테이블에는 포함 + **각주로 drift 명시**. volt [#49](https://github.com/coseo12/volt/issues/49) "주석-구현 drift 는 버그 생성원" 경계:
- drift 자체가 문제가 아니라 **drift 를 인지하지 못하는 상태** 가 문제
- 각주 "CI 자동 실행 미구현 — 다운스트림이 직접 step 추가" 로 전환하면 허용 가능한 drift

#### 8. 리뷰어 vs cross-validator 의 발견 분담 관찰

같은 PR 을 reviewer (Claude sub-agent) 과 Gemini cross-validate 가 독립 검토했을 때 발견 영역이 **분담**:
- **reviewer**: 로직 정확성 (상위 lock 조건 / pytest 의존성 / python-version-file 실패 / cargo --release) + 일관성 (SKILL.md drift / 주석 복잡도) → 7 권고
- **Gemini**: 보안 (`curl | sh` 공급망) + 엣지 케이스 요약 → 1 고유 발견

겹침은 거의 없음 — reviewer 는 코드베이스 컨벤션·과거 volt 연쇄를 잘 보고, Gemini 는 외부 툴 동작·보안·공급망을 잘 본다. **두 관점 병행이 가치** (volt [#55](https://github.com/coseo12/volt/issues/55) Claude 편향 4종 체크리스트의 연장선).

## 교훈 / 다음에 적용할 점

1. **CI 다언어 템플릿은 "lock 우선순위 + 배타 조건" 패턴을 기본** — 단일화 유혹에 volt #51 가드 적용
2. **echo-only step 은 volt #48 트랩** — "감지만" 하는 step 은 실행까지 연결하거나 삭제. 중간 상태 유지 금지
3. **암묵 전제 명시 설치** — pytest / babel / rspec 같은 "dev-dependencies 에 있겠지" 전제는 깨진다. 감지 후 명시 설치 방어
4. **공급망 보안**: `curl | sh` / `wget -O- | bash` 패턴은 CI 에서 금지. 공식 action 또는 checksum 검증 필수
5. **의도적 drift 는 각주로** — volt #49 drift 경계 준수
6. **reviewer + cross-validator 병행** — 발견 영역 분담으로 단일 모델 편향 + 단일 체크리스트 blindspot 모두 해소

## 관련 노트/링크

- harness PR [#178](https://github.com/coseo12/harness-setting/pull/178) (구현 원본)
- harness PR [#179](https://github.com/coseo12/harness-setting/pull/179) (CHANGELOG)
- harness PR [#180](https://github.com/coseo12/harness-setting/pull/180) (release)
- harness release [v2.29.0](https://github.com/coseo12/harness-setting/releases/tag/v2.29.0)
- 선행 volt [#48](https://github.com/coseo12/volt/issues/48), volt [#49](https://github.com/coseo12/volt/issues/49), volt [#51](https://github.com/coseo12/volt/issues/51), volt [#54](https://github.com/coseo12/volt/issues/54), volt [#55](https://github.com/coseo12/volt/issues/55)
