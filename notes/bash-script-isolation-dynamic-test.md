---
title: bash 스크립트 격리 동적 테스트 패턴 — mktemp + env override 4~5 케이스 매트릭스
type: knowledge
source_repo: coseo12/astro-simulator
source_issue: 98
captured_at: 2026-05-17
status: refined
tags: [bash, testing, dynamic-test, mktemp, environment-isolation, verify-script]
related:
  - ./macos-bash-3-2-associative-array.md
  - ../meta/patterns/regression-guard-3stage-simulation.md
  - ./work-order-grep-backtick-fidelity.md
---

## 요약

`verify-*.sh` 류 SSoT 가드 스크립트는 보안/회귀 가드 성격이라 **negative case 검증이 본질**. 정적 분석/lint 만으로는 가드 로직 자체 작동 보장 불가. `mktemp -d` 격리 + `AGENT_DIR` (또는 동종 env var) override + 4~5 케이스 매트릭스로 동적 입증.

## 본문

**패턴 (verify-create-pr-ssot.sh 도입 시 박제)**:

```bash
set -e
TMPDIR=$(mktemp -d -t verify-XXXXXX)
trap "rm -rf $TMPDIR" EXIT
cp -r .claude/agents "$TMPDIR/agents"

# 케이스 1: 정상 (PASS 기대)
AGENT_DIR="$TMPDIR/agents" bash scripts/verify-X.sh

# 케이스 2: SSoT 키 라인 제거 (FAIL 기대)
sed -i.bak '/SSoT 키 패턴/d' "$TMPDIR/agents/pm.md"
AGENT_DIR="$TMPDIR/agents" bash scripts/verify-X.sh || true

# 케이스 3: SSoT 키 한 글자 변경 (FAIL 기대)
cp .claude/agents/pm.md "$TMPDIR/agents/pm.md"  # 원본 복구
sed -i.bak 's/스킬 사용/스킬 호출/' "$TMPDIR/agents/qa.md"
AGENT_DIR="$TMPDIR/agents" bash scripts/verify-X.sh || true

# 케이스 4: 파일 부재 (FAIL 기대)
cp .claude/agents/qa.md "$TMPDIR/agents/qa.md"
rm "$TMPDIR/agents/developer.md"
AGENT_DIR="$TMPDIR/agents" bash scripts/verify-X.sh || true

# 케이스 5: SSoT 키 유지 + 부수 텍스트 drift (FAIL drift detect 기대)
cp .claude/agents/developer.md "$TMPDIR/agents/developer.md"
sed -i.bak 's/(#471)/(#999)/' "$TMPDIR/agents/reviewer.md"
AGENT_DIR="$TMPDIR/agents" bash scripts/verify-X.sh || true
```

**스크립트 측 박제** (verify-*.sh 작성 시 필수):
```bash
# 테스트 격리용 — 기본 .claude/agents 외 경로 검사 시 AGENT_DIR override
AGENT_DIR="${AGENT_DIR:-${PROJECT_DIR}/.claude/agents}"
```

**4~5 케이스 매트릭스 의의**:

| 케이스 | 검증 대상 |
|--------|-----------|
| 1. 정상 | false-positive 부재 (정상을 정상으로 분류) |
| 2. SSoT 라인 완전 제거 | 누락 감지 분기 |
| 3. SSoT 키 자체 변형 | 누락 감지 분기 (키 매칭 변형 흡수) |
| 4. 파일 부재 | 파일 시스템 분기 |
| 5. SSoT 키 유지 + 부수 텍스트 drift | drift 감지 분기 (동일성 검증) |

**자명함정 회피**:
- **케이스 5 박제 의무** — 케이스 2/3 만 검증하면 "키 변경 = 누락" 분기만 검증되고 "부수 텍스트 drift" 분기 (실제 운영 빈도 1순위) 미검증
- **trap EXIT cleanup 의무** — 케이스 도중 fail 해도 TMPDIR 정리 (set -e 와 조합 시 trap 이 안전망)
- **set +e / || true 박제** — fail 기대 케이스에서 종료 코드 활용 + 검증 누락 없이 다음 케이스 진행

**적용 조건**:
- verify-*.sh / SSoT 가드 / 보안 가드 / drift 감지 신규 스크립트
- `AGENT_DIR` / `CHECK_DIR` / `BASE_DIR` 등 env var override 가능한 설계 선행 의무

## 관련 노트/링크

- 사례: astro-simulator PR #496 (#480 5 페르소나 .md create-pr SSoT drift 자동 가드, 2026-05-17) 5/5 PASS 실측
- 선례: `scripts/verify-agent-ssot.sh` (harness #145, AGENT_DIR override 패턴 첫 도입)
- 관련 volt: #95 (macOS bash 3.2 호환), #96 (3중 시뮬레이션 — 격리 동적 테스트와 직교 박제)
