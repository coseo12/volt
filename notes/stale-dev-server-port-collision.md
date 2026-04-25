---
title: stale dev 서버 포트 점유 오진 — sub-agent background cleanup 누락
type: report
source_repo: coseo12/astro-simulator
source_issue: 46
captured_at: 2026-04-19
status: refined
tags: [dev-server, sub-agent, background-process, cleanup, lsof, nextjs, hmr]
related:
  - ../meta/patterns/sub-agent-finalization-miss-pattern.md
  - ./lint-staged-gitignore-silent-revert.md
  - ./reviewer-qa-gate-skip-headless-self-pass.md
---

## 배경/상황

P8 PR-3 구현 중 QA sub-agent 가 `/ko?bh=2` panel 이 visible=false 라고 보고. ADR L18-20 에 따르면 `solar-system.json` 의 moon 엔티티 + `parentId` 체인으로 자동 렌더돼야 함. 즉 **회귀**가 의심됨.

원인 진단:
1. 처음에 `useState+useEffect+window.location.search` → `useQueryState('bh')` nuqs 전환 회귀 가능성 조사 → 정상
2. dev 서버 fresh 재기동 후에도 동일 증상
3. `lsof -i :3001 -P -n` 실행 → 다른 프로세스가 점유 중 (PID 다름)
4. **실제 원인**: 이전 QA sub-agent 가 다른 worktree (`/tmp/astro-simulator-qa-243/`) 에서 띄운 dev 서버가 3001 점유 → 내 `pnpm dev` 가 조용히 3001 에 bind 실패하거나 오래된 bundle 서빙
5. 해당 PID kill 후 feature 브랜치에서 재기동 → 포보스/데이모스 정상 렌더

## 내용

**오진 패턴**:
- sub-agent 가 `run_in_background=true` 로 띄운 dev 서버는 sub-agent 종료 후에도 **살아있다** (Claude Code 의 background task 는 session 단위가 아니라 시스템 프로세스)
- 메인 오케스트레이터가 새 feature 브랜치에서 dev 서버를 띄울 때 같은 포트를 쓰면 충돌 또는 stale 서빙 발생
- 특히 Next.js HMR 이 worktree 경로가 다르면 재컴파일 안 됨 → 최신 코드 변경 반영 안 된 브라우저 테스트

**검출 루틴** (경험 기반):
```bash
# 1. 포트 점유 확인
lsof -i :3001 -P -n
# 2. 다른 cwd 에서 띄운 것이면 해당 PID 의 OPEN 파일 경로 확인
lsof -p <PID> | grep -E 'cwd|\.next/'
# 3. kill 후 재기동
kill <PID>
```

**해결책 후보**:
- (a) 메인 오케스트레이터가 sub-agent 호출 전후로 `lsof -i :<port>` 체크 + stale 프로세스 정리 루틴
- (b) dev 서버 HUD 에 브랜치/SHA 표시 (예: 화면 모서리에 `main@abc1234`) — 육안으로 stale 감지 가능
- (c) sub-agent 프롬프트에 "dev 서버 종료" 를 **마무리 체크리스트 필수 항목**으로 명시

## 교훈 / 다음에 적용할 점

- **sub-agent 마무리 체크리스트 확장**: volt #24 "sub-agent 박제 누락" 규약에 `dev_server_killed: boolean` 필드 추가. 마무리 보고 JSON 에 명시
- **포트 점유 pre-check**: 메인이 `run_in_background=true` 로 서버 띄우기 전에 `lsof -i :<port>` 먼저 확인. 점유 시 kill 여부 사용자에게 확인
- **dev 서버 HUD 개선 (하네스 차원 개선 후보)**: Next.js middleware 로 현재 브랜치/SHA 를 페이지 하단에 inject. 브라우저 검증 시 "최신 코드인가?" 를 즉시 판별
- **관련 선행 교훈**: volt #13 ("커밋 성공 ≠ 의도한 변경 커밋됨") 의 **실행 환경 버전**. "서버 시작됨 ≠ 최신 코드 서빙" 을 체크리스트화
- **오진 비용**: 이번 사례에서 약 15분 추가 (브라우저 재검증 + 원인 수색). 누적되면 sub-agent 파이프라인의 실질 throughput 저하
