---
name: ingest-volt
description: |
  coseo12/volt 레포에 캡처된 capture+knowledge/report 라벨 GitHub 이슈를
  inbox → refine → observations → close 3커밋 워크플로로 정제 박제한다.
  capture-volt 스킬(외부 → 볼트 이슈 생성)과 대칭 방향 — 이쪽은 볼트 내부에서 정제.
  TRIGGER when: /volt-record 커맨드 호출, "이슈 처리", "인박스 정제", "배치 처리" 요청.
  DO NOT TRIGGER when: 외부 프로젝트에서 볼트로 이슈 생성(capture-volt 사용),
  볼트 외 레포 이슈 처리, 단순 이슈 조회.
---

# Volt 이슈 정제 (ingest-volt)

`coseo12/volt` 레포의 capture 이슈를 볼트 파일 구조(`notes/`, `meta/{decisions|feedback|patterns}/`)로 박제한다. 워크플로는 방금 수행한 9차 배치(#27)까지의 운영 규약을 그대로 따른다.

## 사전 조건

- 현재 작업 디렉토리가 `coseo12/volt` 레포 로컬 클론 (`git remote -v` 로 `coseo12/volt.git` 확인).
- `gh` CLI 로그인 완료.
- 작업트리 clean — 수동 변경과 혼합 금지.
- `AGENTS.md` 규약이 이 스킬의 전제. 헤더명·라벨·제목 접두사가 정확히 일치해야 파싱 가능.

## 분류 → 목적지 매핑

| 이슈 label | report_type | frontmatter `type` | 목적지 디렉토리 |
|---|---|---|---|
| knowledge | — | `knowledge` | `notes/` |
| report | troubleshooting | `report` | `notes/` |
| report | retrospective | `report` | `notes/` |
| report | research | `report` | `notes/` |
| report | decision | `decision` | `meta/decisions/` |
| report | feedback | `feedback` | `meta/feedback/` |
| report | pattern | `pattern` | `meta/patterns/` |

**정규화 규칙**: `report + (decision|feedback|pattern)` 은 frontmatter `type` 을 report_type 값으로 치환 (기존 `type: report` + 별도 `report_type:` 필드 대신). 6차 배치(#22)에서 확정.

**주의**: 제목에 "패턴" 단어가 포함돼도 이슈 **라벨**이 `knowledge` 면 `notes/` 로 이동 (9차 배치 #27). 제목이 아닌 라벨·report_type 으로 판단.

## 파일 변환 규칙

### slug

- 영어 kebab-case + `.md` 확장자.
- 이슈 제목에서 핵심 영문 키워드 추출. 한글-only 제목이면 태그·주제 영문으로 대체.
- 충돌 확인: `ls inbox/ notes/ meta/decisions/ meta/feedback/ meta/patterns/` 에 동일 slug 없는지.
- 기존 사례: `lint-staged-gitignore-silent-revert`, `cross-validate-after-policy-freeze`, `manifest-state-recovery-pattern`, `adr-prime-variant-pattern`, `babylon-postprocess-alpha-silent-failure`.

### inbox 단계 frontmatter

```yaml
---
title: {이슈 제목에서 "[knowledge] " 또는 "[report] " 접두사 제거}
type: {knowledge|report|decision|feedback|pattern}  # 정규화 규칙 적용
source_repo: {이슈 본문 "### 출처 레포" 값 — 생략 시 _No response_ 였으면 필드 자체 생략}
source_issue: {이슈 번호, 정수}
captured_at: {오늘 YYYY-MM-DD}
status: inbox
tags: [{이슈 본문 "### 태그" 의 쉼표분리값을 yaml 리스트로}]
related:
  - {참조 대상 경로}  # inbox/ 기준 상대경로, refine 시 최종 목적지 기준으로 보정
---
```

### 본문 헤더 변환

이슈 본문의 `### ` 헤더를 `## ` 로 한 단계 내린다. **단, frontmatter 로 올라간 항목은 본문에서 제거**한다.

**knowledge 타입**:
- 제거: `### 출처 레포`, `### 태그` (frontmatter 로 이동)
- 유지·변환: `### 요약` → `## 요약`, `### 본문` → `## 본문`, `### 관련 노트/링크` → `## 관련 노트/링크`

**report 타입**:
- 제거: `### 리포트 유형`, `### 출처 레포`, `### 태그` (frontmatter 로 이동)
- 유지·변환: `### 배경/상황` → `## 배경/상황`, `### 내용` → `## 내용`, `### 교훈 / 다음에 적용할 점` → `## 교훈 / 다음에 적용할 점`

본문 내부의 `#### ` 서브헤더는 유지. 코드블록·링크·bullet 등 나머지 마크다운은 원문 그대로.

## related 상대경로 규칙

**inbox 단계** (파일이 `inbox/` 에 있을 때):
- 참조 대상이 `notes/X.md` → `../notes/X.md`
- 참조 대상이 `meta/patterns/X.md` → `../meta/patterns/X.md`
- 참조 대상이 `meta/decisions/X.md` → `../meta/decisions/X.md`

**refine 후** (최종 목적지 기준):
- notes/A ↔ notes/B → `./B.md`
- notes/A ↔ meta/patterns/B → `../meta/patterns/B.md`
- meta/patterns/A ↔ notes/B → `../../notes/B.md`
- meta/patterns/A ↔ meta/patterns/B → `./B.md`
- meta/decisions/A ↔ meta/patterns/B → `../patterns/B.md`

8차 배치(#26)에서 `../notes/...` 를 `meta/patterns/` 기준으로 그대로 쓴 실수 발생. **refine 시 반드시 최종 위치 기준으로 재계산**.

## 역참조 백필

새 노트의 `related` 가 기존 refined 노트를 참조하면, **그 기존 노트의 `related` 배열에도 새 노트 링크를 추가**한다. frontmatter 만 수정하고 본문은 건드리지 않는다. 7차 배치(#22)에서 운영 규약화, 8차(#19/#20)·9차(#13)에서 연속 적용.

## 3커밋 워크플로

### 커밋 1 — inbox 캡처

```bash
git add inbox/{slug}.md
# 여러 이슈 배치면 여러 파일 동시 add
git commit -m "$(cat <<'EOF'
chore(inbox): capture issue #{N} into inbox/

{배치 요약 1~2줄 — 주제 도메인/출처/건수}:
- #{N} [{label_or_report_type}] {핵심 제목}

related 그래프 (refine 단계에서 역참조 백필 예정):
- #{N} → {참조 대상 설명}

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

배치 여러 건이면 제목은 `capture issues #{A}-#{B} into inbox/` 또는 `capture issue #{A}, #{B} into inbox/`.

### 커밋 2 — refine (inbox → 목적지)

```bash
git mv inbox/{slug}.md {target_dir}/{slug}.md
# frontmatter 수정: status: inbox → refined, related 경로 보정
# 역참조 백필: 기존 노트의 related 배열 갱신
git add -A
git status   # 의도한 파일만 stage 됐는지 확인
git commit -m "$(cat <<'EOF'
refactor(refine): move #{N} to {target_dir}/ with status: refined

- inbox/{slug}.md → {target_dir}/ ({type})
- frontmatter status: inbox → refined
- related 경로 보정: {요약}
- 역참조 백필: {기존 노트} 의 related 에 {slug} 추가 ({양방향 연결 설명})

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

배치 여러 건이면 제목에 `move #{A}-#{B} to ...` + `backfill ...` 형태로 묶는다 (7차·8차 예시 참고).

### 커밋 3 — observations 로그

`meta/operations/observations.md` 하단 "## 판단 재료 누적 현황" **바로 위**에 배치 섹션 삽입.

**배치 번호 결정**: 파일에서 가장 최근 `## {date} — {N}차 배치` 헤더의 N 을 찾아 +1. (2026-04-18 기준 9차가 최신 → 다음은 10차.)

배치 섹션 템플릿:

```markdown
---

## {YYYY-MM-DD} — {N}차 배치(#{X}{~#Y 또는 단일})

**처리량**: {K}건 ({분류 breakdown, 예: "knowledge 1" 또는 "report·pattern 2 / knowledge 1"}) — `{목적지 요약, 예: "notes/ 1건 + meta/patterns/ 2건"}`

**관찰**:
- {이번 배치에서 관찰된 특이사항 bullet}
- 본문 교정 {M}건. frontmatter 정규화는 refine 에 흡수.

**시그널**:
- {중장기 관찰 — 자동화 트리거, 운영 규약 개정 후보, 누적 지표의 의미 변화}
```

이어서 "## 판단 재료 누적 현황" 표의 행별 업데이트:

| 행 | 갱신 방법 |
|---|---|
| 누적 처리 건수 | +K |
| 본문 교정 발생률 | 분모 갱신, 분자 누적 |
| frontmatter 교정 발생률 | 분모 갱신 |
| 헤더 포맷 변형 | 분모 갱신 |
| 정제 지연(inbox 체류) | 당일/익일/2일+ 해당 버킷 +K |
| 출처 레포 다양성 | 신규 출처 있으면 +1, 비고에 배치 번호 |
| 관련 노트 상호 참조 | 삼각/동타입쌍/교차타입쌍 증분 |
| 역참조 백필 건수 | 백필 건수 +M |
| 볼트 자기참조 노트 건수 | 볼트 내부 노트 참조 시 +1 |
| 분류별 누적 | 해당 타입 +1 |

판단 트리거 네 항목(월 캡처량 / 본문 교정률 / 정제 지연 / 출처 다양성)은 현 수치로 갱신하되, 4월 캡처량 트리거는 이미 충족 상태 유지.

```bash
git add meta/operations/observations.md
git commit -m "$(cat <<'EOF'
docs(ops): log batch #{N} observations ({YYYY-MM-DD})

{N}차 배치({분류 breakdown}) 처리 관찰:
- {핵심 관찰 2~4줄}

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

## 이슈 close

3커밋 완료 후 이슈마다:

```bash
gh issue comment {N} --repo coseo12/volt --body "정제 완료 → {target_dir}/{slug}.md (commit {refine-commit-hash-7자}). {특이 연결사항 또는 생략}."
gh issue close {N} --repo coseo12/volt
```

`{refine-commit-hash}` 는 커밋 2의 앞 7자 해시. `{특이 연결사항}` 예: "volt #13과 교차타입 쌍 연결 — 볼트 자기참조 노트 첫 사례" (없으면 문장 자체 생략).

## 민감정보 스캔

본문 변환 전 확인:
- 비밀키/토큰 (`sk-`, `ghp_`, `AKIA`, `-----BEGIN` 등 패턴).
- 내부 URL (사내 도메인, 프라이빗 IP).
- 고객 데이터 (이메일, 실명, 개인식별정보).

발견 시 중단하고 사용자에게 보고.

## 관찰 초안 작성 가이드

배치마다 기록할 **관찰** 항목의 재료:
- 해당 배치에서 처음 발생한 사건(첫 사례)이 있는가?
- 역참조 백필·상대경로 보정 등 refine 과정에서 눈에 띈 것은?
- 분류 판단이 애매했거나 제목/라벨 불일치가 있었는가?
- 본문 교정이 필요했는가? (이슈 본문 원문을 얼마나 수정했는가)

**시그널** 항목의 재료:
- 반복되는 특징이 운영 규약 개정으로 이어질 여지가 있는가?
- 자동화 판단 트리거 지표(캡처량·교정률·정제지연·출처다양성) 중 움직인 것이 있는가?
- 장기적으로 볼트의 스코프·성격이 변하고 있다는 신호는?

관찰은 사실, 시그널은 해석 — 섞지 않는다.

## 금지/주의

- 3커밋 분할 생략 금지 (이력·rollback).
- frontmatter 헤더명 임의 변경 금지 (파싱 규약).
- 같은 slug 중복 금지 (충돌 확인 필수).
- 정제와 무관한 파일 변경을 같은 커밋에 섞지 말 것.
- `git add -A` 전 반드시 `git status` 로 의도한 변경만 stage 됐는지 확인.
- 민감정보 포함 이슈는 캡처 중단 후 사용자 보고 (public 레포).

## 참고

- [AGENTS.md](../../../AGENTS.md) — 이슈 포맷 규약(헤더명 의존)
- [meta/operations/observations.md](../../../meta/operations/observations.md) — 배치 로그 이력, 운영 규약 진화 기록
- [notes/volt-slash-command-and-skill.md](../../../notes/volt-slash-command-and-skill.md) — 반대 방향 페어(`/volt` + `capture-volt`, 외부 → 볼트 이슈 생성)
