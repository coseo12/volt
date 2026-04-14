---
title: Claude Code /volt 슬래시커맨드 + capture-volt 스킬 이식 가이드
type: knowledge
source_repo: coseo12/astro-simulator
source_issue: 8
captured_at: 2026-04-14
status: refined
tags: [claude-code, skill, slash-command, volt, portability]
related:
  - https://github.com/coseo12/volt/blob/main/AGENTS.md
  - https://github.com/coseo12/volt/blob/main/README.md
  - https://github.com/coseo12/volt/blob/main/CONTRIBUTING.md
---

## 요약

Claude Code에서 `/volt` 한 번으로 현재 대화의 지식/경험을 `coseo12/volt` 레포 이슈로 캡처하는 슬래시커맨드 + 스킬 페어. 어느 프로젝트에서든 `.claude/commands/volt.md`와 `.claude/skills/capture-volt/SKILL.md` 두 파일만 넣으면 동일 동작한다. 기존에는 `~/.claude`(글로벌)에만 두었지만, 프로젝트 단위로 버전 관리하면 팀/에이전트 온보딩이 간단해진다.

## 본문

### 설치 (프로젝트 로컬)

```bash
# 레포 루트에서
mkdir -p .claude/commands .claude/skills/capture-volt
# 아래 두 파일 내용을 복사해 저장
$EDITOR .claude/commands/volt.md
$EDITOR .claude/skills/capture-volt/SKILL.md

# 최초 1회 — volt 레포 라벨 보장
gh label create capture   --repo coseo12/volt --color "0E8A16" --description "캡처된 이슈" || true
gh label create knowledge --repo coseo12/volt --color "1D76DB" --description "재사용 가능한 지식" || true
gh label create report    --repo coseo12/volt --color "5319E7" --description "작업 리포트" || true
```

### 설치 (글로벌 — 단일 사용자 기본값으로 두고 싶을 때)

경로만 `~/.claude/commands/volt.md`, `~/.claude/skills/capture-volt/SKILL.md`로 바꾼다. 단, 프로젝트 로컬과 글로벌에 같은 이름이 공존하면 Claude Code가 중복 등록하므로 한쪽만 둘 것.

### 동작 요약

1. 사용자가 `/volt` 입력 (옵션: 주제 힌트 `$ARGUMENTS`).
2. 커맨드가 `capture-volt` 스킬을 로드 → AGENTS.md 규약(라벨·제목·헤더명) 컨텍스트 확보.
3. 최근 대화에서 캡처 후보 선정(또는 힌트 범위 검색) → `knowledge` vs `report+세부유형` 분류.
4. 민감정보 스캔 + 중복 이슈 검색(`gh issue list --repo coseo12/volt --search`).
5. 고정 헤더 템플릿으로 `gh issue create` 실행 → 볼트 관리자가 해당 이슈를 `notes/` 또는 `meta/*`로 정제 이동.

### `.claude/commands/volt.md` (전체)

````markdown
---
description: 재사용 가능한 지식/경험/결정을 coseo12/volt 레포에 이슈로 캡처 (RAG 원천)
argument-hint: [주제 힌트 — 생략 시 최근 대화에서 가치 있는 것을 제안]
allowed-tools: [Bash, Read, Skill]
---

# /volt — Volt 캡처

coseo12/volt 레포에 GitHub Issue로 지식/경험을 캡처한다. AGENTS.md 규약(라벨·제목 접두사·본문 헤더명)을 따른다.

상세 절차는 **`capture-volt` 스킬**에 정의돼 있으니 먼저 그 스킬을 호출한다.

## 사용자 입력

`$ARGUMENTS`

## 실행 절차

1. **스킬 호출** — 먼저 `capture-volt` 스킬을 실행해 규약·템플릿·민감정보 경고를 로드한다.

2. **대상 선정**:
   - `$ARGUMENTS`에 주제 힌트가 있으면 그 범위에서 최근 대화를 뒤져 1건 선정.
   - 비어있으면 최근 대화에서 캡처 가치가 있는 후보(1~3건)를 bullet로 제시하고 사용자에게 선택 요청.
   - 후보는 재사용 가능한 형태여야 함 — 프로젝트 세부 구현보다 **교차 프로젝트에서 유효한 교훈·패턴·결정**.

3. **분류 결정**:
   - 시간·맥락 독립 개념/패턴/도구 사용법 → `knowledge`
   - 특정 작업의 경험/결정/회고 → `report` + 세부 유형(troubleshooting/retrospective/research/decision/feedback/pattern)
   - 애매 시 `report` + `research` 폴백.

4. **민감정보 스캔** — 본문에 비밀키·내부 URL·고객 데이터 없는지 확인. public 레포.

5. **중복 확인**:
   ```bash
   gh issue list --repo coseo12/volt --search "<키워드>" --state all
   ```

6. **이슈 생성** — capture-volt 스킬의 템플릿(`gh issue create --repo coseo12/volt --title "[knowledge|report] ..." --label "capture,knowledge|report" --body ...`)을 그대로 사용. 헤더명은 절대 바꾸지 않는다 (볼트 규약이 정확히 이 문자열에 의존).

7. **결과 보고** — 생성된 이슈 URL을 사용자에게 표시. 2건 이상이면 표로 정리.

## 자주 놓치는 규약

- 라벨은 **반드시 두 개**: `capture` + (`knowledge` 또는 `report`). 누락 시 이슈 생성 실패.
- 제목 접두사 `[knowledge] ` 또는 `[report] ` (대괄호, 소문자, 뒤 공백 1칸).
- 본문 헤더는 `### 출처 레포` / `### 태그` / `### 요약` / `### 본문` / `### 관련 노트/링크` (knowledge) 또는 `### 리포트 유형` / `### 출처 레포` / `### 태그` / `### 배경/상황` / `### 내용` / `### 교훈 / 다음에 적용할 점` (report). 공백·기호 포함 정확히.
- 선택 필드에 내용 없으면 `_No response_` 또는 섹션 생략.

## 금지

- 민감정보(비밀키, 토큰, 내부 URL, 고객 데이터) 포함.
- 헤더명 임의 변경.
- 여러 주제를 한 이슈에 묶기 — 1건 1주제 원칙.
- 사용자가 명시 요청하지 않은 캡처를 한꺼번에 대량 생성.
````

### `.claude/skills/capture-volt/SKILL.md` (전체)

````markdown
---
name: capture-volt
description: |
  재사용 가능한 지식/경험/의사결정을 coseo12/volt 레포의 GitHub Issue로 캡처하여
  RAG 원천 데이터로 축적하는 스킬. AGENTS.md 규약(라벨·제목 접두사·본문 헤더명)을 따른다.
  TRIGGER when: 세션 중 agent/skill 개선에 유용한 교훈·패턴·결정이 발생했을 때,
  회고를 남겼을 때, 사용자가 "볼트에 남겨", "volt 캡처", "knowledge 기록", "volt에 넘겨" 등을 요청했을 때.
  DO NOT TRIGGER when: 프로젝트 내부 이슈/PR 생성일 때(create-issue 사용), 민감정보 포함 콘텐츠일 때.
---

# Volt 캡처

`coseo12/volt`는 Claude Code agent/skill 개선용 RAG 원천 데이터 저장소(공개).
이 스킬은 AGENTS.md 규약에 맞춰 `gh issue create`로 이슈를 생성한다.

## 사전 조건

1. 레포는 public — **민감정보(비밀키, 내부 URL, 고객 데이터) 절대 금지**.
2. 라벨이 없으면 실패. 첫 사용 시 아래 명령 실행:

```bash
gh label create capture   --repo coseo12/volt --color "0E8A16" --description "캡처된 이슈" || true
gh label create knowledge --repo coseo12/volt --color "1D76DB" --description "재사용 가능한 지식" || true
gh label create report    --repo coseo12/volt --color "5319E7" --description "작업 리포트" || true
```

## 분류 결정 트리

```
캡처할 내용이 무엇인가?
│
├─ 시간·맥락 독립, 재사용 가능 개념/패턴/도구 사용법
│   → knowledge                                  (→ notes/)
│
└─ 특정 작업의 경험·결과·결정
    → report + report_type 중 하나:
        ├─ troubleshooting : 문제 해결 기록       (→ notes/)
        ├─ retrospective   : 회고                (→ notes/)
        ├─ research        : 조사/리서치          (→ notes/)
        ├─ decision        : 도구/접근법 선택 근거  (→ meta/decisions/)
        ├─ feedback        : AI 동작 피드백        (→ meta/feedback/)
        └─ pattern         : 반복 관찰된 워크플로 패턴 (→ meta/patterns/)
```

판단 애매 시 폴백: `report` + `research`.

## 중복 방지

이슈 생성 전 반드시 유사 이슈 확인:

```bash
gh issue list --repo coseo12/volt --search "키워드" --state all
```

## 본문 구조 (헤더명 고정 — 볼트 규약이 의존)

### knowledge 타입

```markdown
### 출처 레포

{owner}/{repo}   (또는 생략)

### 태그

tag1, tag2

### 요약

한 문단 핵심 요약.

### 본문

상세 내용.

### 관련 노트/링크

- URL 또는 노트명
```

### report 타입

```markdown
### 리포트 유형

troubleshooting | retrospective | research | decision | feedback | pattern

### 출처 레포

{owner}/{repo}

### 태그

tag1, tag2

### 배경/상황

어떤 작업 중 어떤 문제/결정이 있었는가.

### 내용

시도·관찰·결론·교훈.

### 교훈 / 다음에 적용할 점

agent/skill 개선에 반영할 포인트.
```

선택 필드에 내용이 없으면 `_No response_` 또는 섹션 생략.

## 제목 규약

- 접두사 고정: `[knowledge] ` 또는 `[report] ` (대괄호 포함, 소문자, 뒤 공백 1칸)
- 본체: 핵심을 한 문장으로, 가급적 60자 이내

## gh 명령 템플릿

### knowledge

```bash
gh issue create \
  --repo coseo12/volt \
  --title "[knowledge] {핵심 요약}" \
  --label "capture,knowledge" \
  --body "$(cat <<'EOF'
### 출처 레포

{owner}/{repo}

### 태그

{comma-separated}

### 요약

{한 문단}

### 본문

{상세}

### 관련 노트/링크

- {URL or note}
EOF
)"
```

### report

```bash
gh issue create \
  --repo coseo12/volt \
  --title "[report] {핵심 요약}" \
  --label "capture,report" \
  --body "$(cat <<'EOF'
### 리포트 유형

{troubleshooting|retrospective|research|decision|feedback|pattern}

### 출처 레포

{owner}/{repo}

### 태그

{comma-separated}

### 배경/상황

{배경}

### 내용

{시도·관찰·결론}

### 교훈 / 다음에 적용할 점

{takeaway}
EOF
)"
```

## 절차

1. **분류 결정**: knowledge / report + 세부 유형 확정.
2. **민감정보 스캔**: 본문에 비밀키·내부 URL·고객 데이터 없는지 확인.
3. **중복 검색**: `gh issue list ... --search`로 유사 이슈 확인.
4. **이슈 생성**: 위 템플릿으로 `gh issue create` 실행.
5. **생성 URL 사용자에게 보고**.

## 금지/주의

- 헤더명(`### 리포트 유형` 등) 변경 금지 — 볼트 규약이 정확히 이 문자열에 의존.
- 라벨·제목 접두사 생략 금지.
- 민감정보 포함 금지 (public 레포).
- 본 스킬은 **이슈 생성까지만** 책임짐. 파일 변환·정제는 볼트 측 워크플로 몫.

## 참고

- [volt README](https://github.com/coseo12/volt/blob/main/README.md)
- [volt AGENTS.md](https://github.com/coseo12/volt/blob/main/AGENTS.md) — 이 스킬의 원본 규약
- [volt CONTRIBUTING.md](https://github.com/coseo12/volt/blob/main/CONTRIBUTING.md)
````

### 주의

- 본 레포는 public. 비밀키/토큰/내부 URL/고객 데이터 절대 금지.
- 헤더명(`### 출처 레포`, `### 태그`, `### 요약` 등)은 볼트 규약. 임의 변경 금지.
- 라벨은 반드시 두 개(`capture` + `knowledge|report`). 누락 시 이슈 생성 실패.

## 정제 시 교정 사항 (원본 이슈 대비)

- "볼트 파서" 표현을 "볼트 관리자 / 볼트 규약"으로 교체 (자동 파서는 아직 없음, 수동 정제 중).
- 분류 트리에 knowledge 및 report의 세부 유형별 최종 목적지(`notes/` 또는 `meta/*`)를 명시 추가.
- `knowledge/` / `reports/` 표기를 실제 구조(`notes/`, `meta/*`)로 교체.
