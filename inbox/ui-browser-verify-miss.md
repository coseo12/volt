---
title: UI 작업에서 브라우저 3단계 검증 누락 사고
type: feedback
source_repo: coseo12/astro-simulator
source_issue: 1
captured_at: 2026-04-14
status: inbox
tags: [process, browser-verify, p1-p2]
related: []
---

## 배경/상황

P1(태양계 MVP) 진행 중 여러 UI PR에서 CRITICAL #3(브라우저 3단계 검증) 위반이 반복됨. 빌드/단위 테스트 PASS만 보고 "동작한다"고 판단한 뒤 커밋했고, 사용자 지적으로 세션 내 교정됨. "E2에서 한 번에 검증한다"는 변명으로 PR별 검증을 미루는 패턴이 드러남.

## 내용

- Group D 구간(9개 UI PR)에서 정적/인터랙션/흐름 3단계 검증 없이 순차 머지.
- 실제 브라우저에서 이벤트 핸들러 누락·URL 동기화 버그가 사후에 드러남.
- 빌드 통과는 렌더링 증거일 뿐, 동작 증거가 아니라는 인식이 반복적으로 풀렸음.
- 교정은 사용자 지적 → CLAUDE.md에 CRITICAL DIRECTIVES 블록 추가 → 각 세션 첫 작업 전 자기 점검 의무화로 이어짐.

## 교훈 / 다음에 적용할 점

- **UI 변경을 포함한 모든 PR에서 브라우저 3단계 증거(스크린샷 또는 Playwright 스크립트 경로)를 PR 본문에 반드시 첨부.** PR 템플릿에 필수 섹션으로 박아두면 자기 점검이 자동화됨.
- "E2로 미룬다" 같은 합리화 문구는 경고 신호. 기능 PR마다 검증이 기본값.
- 빌드·단위 테스트 PASS + 스크린샷은 Level 1에 불과. 인터랙션(클릭/폼)·흐름(URL↔상태) 레벨까지 필수.
- skill/agent 프롬프트에 "빌드 PASS ≠ 동작 PASS" 문구와 3단계 체크 포함.
