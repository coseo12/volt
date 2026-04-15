# AGENTS.md

AI 에이전트(Claude Code 등)가 이 레포에 지식/리포트를 **GitHub Issue로 전달**하기 위한 규약.
사전 지식 없이 이 문서만 읽고도 올바른 이슈를 만들 수 있도록 작성됨.

## 레포

- `coseo12/volt` (public)
- 용도: 개발 지식 및 작업 리포트 축적 → Claude Code agent/skill 개선용 RAG 원천

## 전달 절차 요약

1. 내용의 성격을 아래 **의사결정 트리**로 분류한다.
2. 해당 **라벨 + 제목 접두사 + 본문 구조**로 이슈를 생성한다.
3. `gh issue create` 명령을 사용한다 (아래 템플릿 참고).

## 의사결정 트리

```
전달할 내용이 무엇인가?
│
├─ 재사용 가능한 개념·패턴·도구 사용법 (시간·맥락 독립적)
│   → 타입: knowledge
│   → 라벨: capture, knowledge
│   → 제목: "[knowledge] ..."
│
└─ 특정 작업에서 발생한 경험·결과·결정 (시간·맥락 종속적)
    → 타입: report
    → 라벨: capture, report
    → 제목: "[report] ..."
    → 본문의 "리포트 유형" 필드로 세부 분류:
        ├─ troubleshooting  : 문제·장애 해결 기록
        ├─ retrospective    : 회고
        ├─ research         : 조사·리서치
        ├─ decision         : 도구/접근법 선택 근거 → 볼트에서 meta/decisions/ 로 이동
        ├─ feedback         : AI 동작 피드백     → 볼트에서 meta/feedback/ 로 이동
        └─ pattern          : 반복 관찰된 워크플로 패턴 → 볼트에서 meta/patterns/ 로 이동
```

**판단이 애매할 때 폴백**: `report` + `report_type: research` 를 기본값으로 사용.

## 라벨 규약

이슈에는 항상 **두 개의 라벨**이 붙어야 한다:

- `capture` (공통, 캡처된 이슈임을 표시)
- `knowledge` 또는 `report` (타입)

## 제목 규약

- 접두사: `[knowledge] ` 또는 `[report] ` (대괄호 포함, 소문자, 뒤 공백 1칸)
- 제목 본체: 내용 핵심을 한 문장으로, 가급적 60자 이내

## 본문 구조

본문은 **GitHub 이슈 폼이 생성하는 렌더 구조와 동일하게** `### 필드명` 헤더 블록으로 작성한다.
필드 순서와 헤더명은 정확히 아래와 같아야 한다 (자동 변환 스크립트가 이를 파싱하기 때문).

### knowledge 타입

```markdown
### 출처 레포

owner/repo-name

### 태그

tag1, tag2, tag3

### 요약

한 문단으로 핵심만 요약.

### 본문

상세 내용. 마크다운/코드블록 자유.

### 관련 노트/링크

- https://example.com
- 관련된 기존 노트 파일명
```

선택 필드(출처 레포, 태그, 관련 노트/링크)는 값이 없으면 `_No response_` 를 넣거나 섹션 자체를 생략한다.

### report 타입

```markdown
### 리포트 유형

troubleshooting

### 출처 레포

owner/repo-name

### 태그

tag1, tag2

### 배경/상황

어떤 작업 중 어떤 문제·결정이 있었는가.

### 내용

시도, 관찰, 결론 등.

### 교훈 / 다음에 적용할 점

agent/skill 개선에 반영할 포인트.
```

`리포트 유형` 필드 값은 반드시 다음 중 하나: `troubleshooting`, `retrospective`, `research`, `decision`, `feedback`, `pattern`.

## 이슈 생성 명령 템플릿

### knowledge 예시

```bash
gh issue create \
  --repo coseo12/volt \
  --title "[knowledge] Python asyncio 취소 전파 규칙" \
  --label "capture,knowledge" \
  --body "$(cat <<'EOF'
### 출처 레포

coseo12/some-project

### 태그

python, asyncio

### 요약

asyncio.Task 가 취소될 때 자식 태스크로 취소가 어떻게 전파되는지 정리.

### 본문

...

### 관련 노트/링크

- https://docs.python.org/3/library/asyncio-task.html
EOF
)"
```

### report 예시

```bash
gh issue create \
  --repo coseo12/volt \
  --title "[report] SSE 스트림 끊김 원인 분석" \
  --label "capture,report" \
  --body "$(cat <<'EOF'
### 리포트 유형

troubleshooting

### 출처 레포

coseo12/some-project

### 태그

sse, nginx, timeout

### 배경/상황

프로덕션에서 장시간 SSE 연결이 60초마다 끊기는 이슈 발생.

### 내용

...

### 교훈 / 다음에 적용할 점

...
EOF
)"
```

## 라벨 자동 생성

위 두 라벨(`capture`, `knowledge`, `report`)이 레포에 없다면 이슈 생성이 실패한다.
필요 시 아래 명령으로 미리 생성:

```bash
gh label create capture   --repo coseo12/volt --color "0E8A16" --description "캡처된 이슈" || true
gh label create knowledge --repo coseo12/volt --color "1D76DB" --description "재사용 가능한 지식" || true
gh label create report    --repo coseo12/volt --color "5319E7" --description "작업 리포트" || true
```

## 이후 처리

이슈가 생성되면 볼트 관리자(사람 또는 자동화)가:

1. 이슈 본문을 `inbox/{slug}.md` 파일로 변환
2. frontmatter(`type`, `source_repo`, `source_issue`, `captured_at`, `status: inbox`, `tags`) 부여
3. 정제 단계를 거쳐 `notes/` 또는 `meta/{feedback|patterns|decisions}/` 로 이동, `status: refined` 로 전환

AI 에이전트는 **이슈 생성까지만** 책임진다. 파일 변환·정제는 볼트 쪽 워크플로의 몫이다.

## 사후 보강 (정제 완료 노트의 갱신)

이미 정제되어 볼트에 저장된 노트에 추가 관찰이 생겼을 때의 처리 절차.

### 유입 경로

1. **Closed 이슈에 코멘트 추가** (기본 경로)
   - 원본 이슈와 맥락이 한 자리에 모이므로 단일 진입점으로 유지.
   - `gh issue list` 기본은 open 만 보이므로 검색 시 `--state all` 필요.
2. **PR 로 노트 직접 수정** (작정한 편집이 필요할 때)
   - 섹션 구조를 다듬거나 전체 리팩터링이 필요하면 이쪽.

### 노트 갱신 방법

- 노트 말미에 **`## 보강 (YYYY-MM-DD 후속)`** 섹션을 새로 추가한다 (기존 섹션은 건드리지 않음, 이력 보존).
- 섹션 내에 출처 이슈 번호 / 코멘트 링크 / 커밋 해시를 명시한다.
- Frontmatter 는 변경하지 않는다 — 변경 이력은 git log 가 진실 공급원.

### 이슈 회신

- Closed 이슈에 갱신 커밋 해시 + 파일 경로를 코멘트로 남긴다.
- 이슈는 **재오픈하지 않는다**.

### 별도 노트로 분리 기준

- 보강이 원 주제에서 벗어나 독립 가치를 가지면 새 노트로 승격.
- 원 노트와 새 노트는 `related:` frontmatter 로 양방향 연결.
- 보강이 3회 이상 누적되어 본문이 비대해지면 리팩터링 검토.

### AI 에이전트의 역할

- 이슈 코멘트 추가까지만 책임 (`gh issue comment {N} --repo coseo12/volt --body ...`).
- 노트 파일 수정·커밋은 볼트 관리자의 정제 워크플로 몫.

## 금지/주의

- 본문 섹션 헤더명(`### 출처 레포` 등)을 임의로 바꾸지 않는다 (파싱 깨짐).
- 라벨·제목 접두사를 생략하지 않는다.
- 민감정보(비밀키, 내부 URL, 고객 데이터)를 포함하지 않는다. **이 레포는 public 이다.**
- 중복 이슈 방지를 위해 생성 전 `gh issue list --repo coseo12/volt --search "키워드"` 로 유사 이슈 확인 권장.
