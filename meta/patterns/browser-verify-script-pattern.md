---
title: 마일스톤별 브라우저 3단계 검증 자동화 스크립트 패턴
type: pattern
source_repo: coseo12/astro-simulator
source_issue: 3
captured_at: 2026-04-14
status: refined
tags: [process, playwright, verify, milestone]
related: []
---

## 배경/상황

P2 여러 마일스톤을 거치며 UI 기능마다 Playwright 기반 브라우저 3단계 검증 스크립트를 작성. 회귀 감지와 CRITICAL #3 위반 방지에 효과적이었음. 네이밍/일괄 실행 패턴을 정립.

## 내용

네이밍 규칙:
- 개별 기능: `scripts/browser-verify-<feature>.mjs`
  예: browser-verify-engine-toggle / browser-verify-dwarf-planets / browser-verify-presets
- 마일스톤 종합: `scripts/verify-p<n><letter>.mjs`
  예: verify-p2b / verify-p2c — 개별 스크립트를 `execSync`로 순차 실행 + bench 회귀 체크

각 스크립트 구조:
1. `[1/3] 정적` — 렌더/콘솔에러 0
2. `[2/3] 인터랙션` — 클릭/폼/토글 실제 동작
3. `[3/3] 흐름` — URL 상태 복원/네비게이션/데이터 연동
4. 스크린샷: `.verify-screenshots/<feature>/{1-static,2-interaction,3-flow}.png`
5. 종료 코드: 실패 시 exit 1

마일스톤 종합에서는:
- 개별 스크립트 실패 = 종합 실패
- bench:scene 실행 → baseline 대비 회귀율 비교(자체 허용 기준: P2-B 25%, P2-C 10%)

## 교훈 / 다음에 적용할 점

- **Playwright 기반 verify 스크립트 파일명·구조를 skill 템플릿화**하면 매 기능마다 재작성 비용이 0에 수렴.
- 마일스톤 종료 시점에 "verify-p<n><letter>.mjs 작성 + bench 회귀 체크 + baseline 갱신"을 의무 스텝으로.
- 회귀 허용 기준은 마일스톤 성격(UI만 / 콘텐츠 추가 / 렌더 변경)에 따라 동적으로. 한 번 고정 기준(±2fps)은 실제 변경량 앞에서 무너짐 — 실측 후 재조정하는 이터레이션이 자연스러움.
- skill: `browser-test` 확장 — 기능 이름만 주면 scripts/browser-verify-<name>.mjs 스캐폴딩.
