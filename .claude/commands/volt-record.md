---
description: 볼트에 캡처된 GitHub 이슈를 inbox → refine → observations → close 3커밋 워크플로로 정제 박제
argument-hint: [이슈 번호(선택) — 예 "27", "24-26", 생략 시 OPEN capture 이슈 전체]
allowed-tools: [Bash, Read, Edit, Write, Glob, Grep, Skill]
---

# /volt-record — Volt 이슈 정제 처리

`coseo12/volt` 레포에 캡처된 `capture+{knowledge|report}` 라벨 이슈를 볼트 파일 구조(`inbox/` → `notes/` 또는 `meta/{decisions|feedback|patterns}/`)로 박제한다. 3커밋 분할(inbox 캡처 / refine / observations 로그) + 이슈 close 까지 1회 실행으로 완료.

상세 절차·분류 매핑·커밋 메시지 템플릿·역참조 백필 규칙·observations 로그 포맷은 **`ingest-volt` 스킬**에 정의돼 있다. 먼저 스킬을 호출한다.

## 사용자 입력

`$ARGUMENTS`

## 실행 절차

1. **스킬 호출** — `ingest-volt` 스킬을 실행해 규약을 로드한다.

2. **전제조건 확인**:
   - 현재 작업 디렉토리가 `coseo12/volt` 레포 로컬 클론인지 확인 (`git remote -v`).
   - 작업트리 clean 인지 확인 (`git status`). 수동 변경과 섞이면 중단 후 사용자 확인.

3. **대상 선정**:
   - `$ARGUMENTS` 단일 번호(예: `27`) → 해당 이슈만 처리.
   - 범위(예: `24-26`) → 해당 범위 전체를 하나의 배치로 처리.
   - 비어있음 → `gh issue list --repo coseo12/volt --label capture --state open` 으로 대상 목록을 표시하고 사용자 확인 후 진행.

4. **배치 처리** (이슈마다 순차, 커밋 1·2는 이슈당):
   - 본문 fetch (`gh issue view {N}`) → 분류 결정(스킬의 분류→목적지 표) → 목적지 확정 → slug 결정.
   - 민감정보 스캔 (public 레포).
   - `inbox/{slug}.md` 작성 + **커밋 1** (`chore(inbox): capture issue #{N} into inbox/`).
   - `git mv` 로 목적지 이동 + frontmatter `status: inbox → refined` + `related` 경로 보정 + 기존 노트 역참조 백필 + **커밋 2** (`refactor(refine): move #{N} to {target}/ with status: refined`).

5. **배치 종료** (배치당 1회):
   - `meta/operations/observations.md` 에 신규 배치 섹션 추가 + "판단 재료 누적 현황" 표 행 업데이트 + **커밋 3** (`docs(ops): log batch #{N} observations ({YYYY-MM-DD})`).
   - 각 이슈에 `gh issue comment` (정제 결과 링크) + `gh issue close`.

6. **결과 보고** — 생성된 3커밋 해시, 이동된 파일 경로, close 된 이슈 번호를 사용자에게 표시.

## 자주 놓치는 규약

- 3커밋 분할은 이력 보존·rollback 용이성 목적 — 생략 금지.
- `report + (decision|feedback|pattern)` 은 frontmatter `type` 을 report_type 값으로 정규화 (6차 배치에서 확정).
- 역참조 백필은 frontmatter 의 `related` 배열만 건드리고 본문은 수정하지 않는다 (7차 배치에서 운영 규약화).
- 상대경로는 최종 목적지 기준 — inbox 단계의 `../notes/...` 는 refine 시 보정 필요 (8차 배치 교훈).

## 금지

- 민감정보(비밀키, 내부 URL, 고객 데이터) 포함 노트 생성 — public 레포.
- frontmatter 헤더명(`type`, `source_issue`, `status`, `related` 등) 임의 변경.
- 정제와 무관한 파일 변경을 같은 커밋에 섞기.
- 사용자 승인 없이 동일 배치를 중복 생성.
