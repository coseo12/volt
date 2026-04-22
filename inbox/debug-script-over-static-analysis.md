---
title: 아키텍처 drift 조사에서 debug 스크립트 실측 > 정적 분석 — Explore 미결정을 30초에 확정한 사례
type: knowledge
source_repo: coseo12/astro-simulator
source_issue: 67
captured_at: 2026-04-22
status: inbox
tags: [debug-script, static-analysis, investigation, architecture-drift, explore-agent, runtime-truth]
related:
  - ../notes/comment-contract-implementation-drift.md
  - ../notes/downstream-measurement-final-guard.md
  - ../notes/no-op-adr-handoff-verification.md
  - ../meta/patterns/cross-validate-after-policy-freeze.md
  - ../notes/cross-validate-accept-vs-defer.md
  - ../notes/cross-validate-dod-self-contradiction.md
---

## 요약

아키텍처 근간 drift 조사에서 **정적 분석(read-only Explore, 코드 리뷰) 은 표현 일관성 확인에 강하지만 runtime 실체 추적에는 약하다**. 20~30줄의 일회성 debug 스크립트가 정적 분석으로 미결정된 의심을 30초만에 확정하는 경우가 반복된다. 실측을 "조사 필수 도구" 로 격상하는 규약 제안.

## 본문

**원 사례 (coseo12/astro-simulator#292)**:

1. **의심 단계**: QA 1차에서 "ADR §3 'Heliocentric 절대 좌표' 선언 vs Rust engine `positions()` 실측 ~2000+ AU offset" drift 관찰 (PR #291 의 P11-A Floating Origin)
2. **정적 분석 위임 (Explore 에이전트)**: ADR / `solar-system-loader.ts` / `buildInitialState()` / `NBodyEngine::new` 체인 전체 추적. 결론: **"ADR Heliocentric 선언 ↔ Sun=(0,0,0) 표현상 일관성"** — **(C) 미결정** 판정. 추가 조사 필요 항목으로 "시간 설정 / 엔진 경로 / timeScale" 나열
3. **실측 (debug 스크립트)**: 메인이 `scripts/_debug-coord-tmp.mjs` 20~30줄 작성 — Playwright + pause URL + focus 버튼 클릭 + `__floatingOrigin.originOffset` 실측. 3분 실행으로 **Sun world = (0, −338 AU, 1041 AU)** → 1094 AU drift 관찰 확보
4. **원인 규명**: debug 데이터 기반으로 코드 재탐색 → `solar-system-scene.ts:511` safety net 조건 분기 누락 버그 확정. 수정 1줄 (커밋 `d61f657`)

**실측이 없었다면**:

Explore 판정 "표현 일관성" 을 사용자가 수용할 경우:
- (A) 문서 오류로 오판정 → ADR §3 을 실제와 무관한 좌표계로 재작성
- 또는 (B) Newton engine instability 로 오판정 → 물리 엔진 전면 재조사 착수
- 둘 다 수 시간~수 일 낭비 + 진범(1 줄 구현 버그) 미발견

**정적 분석의 한계 재현**:

Explore 가 놓친 지점: `solar-system-scene.ts:511-525` safety net 이 focus 상태와 무관하게 매 프레임 `floatingOrigin.update(cameraWorldMeters)` 호출하는 코드 블록. 주석(line 507-510)에는 "focus 가 없는 free-fly 탐색 중" 이라 명시됐으나 **구현 조건 분기 누락**. 정적 분석이 주석과 일관성을 인정하는 동안 실체는 다른 경로로 실행됨.

**일반화 가능성**:

- 주석 계약 vs 구현 drift (CLAUDE.md 교훈 / volt #49) 와 동일 구조
- 정적 분석은 **주석과 타입 시그니처** 를 본다. runtime 분기/상태/이벤트 loop 는 보지 못함
- 특히 "조건 분기 추가 금지 (`if` 누락)" 같은 omission 버그는 grep 으로 안 잡힘

## 본문 (세부)

**실측 스크립트 템플릿 (재사용 가능)**:

```js
// 임시: scripts/_debug-<topic>-tmp.mjs (커밋 금지, 즉시 rm)
import { chromium } from 'playwright';
const browser = await chromium.launch();
const page = await browser.newPage();
await page.goto('http://localhost:3000/?<pause params>', { waitUntil: 'networkidle' });
await page.evaluate(() => window.__store?.getState?.()?.setPause?.(true));
const snapshot = await page.evaluate(() => ({ ... }));
console.log(JSON.stringify(snapshot, null, 2));
await browser.close();
```

특징:
- `/tmp/` 에 두면 `node_modules` resolve 실패 → 프로젝트 `scripts/` 하위 `_debug-*-tmp.mjs` 로 둔 후 실행 직후 `rm`
- dev 서버 기동 + Monitor 로 ready 대기 (CLAUDE.md § polling 규약)
- pause / state override URL 파라미터 활용으로 drift 원인과 측정 오차 분리

**규약 제안**:

1. **cross-validate / volt-review / architect 스킬 명시 업데이트** — "아키텍처 근간 drift 조사 시 debug 스크립트 실측 선행 권장" 1 라인 추가
2. **Explore agent 설명 보강** — "runtime 상태/이벤트 loop / 조건 분기 omission 버그는 실측이 보완" 경고 1 라인
3. **harness CLAUDE.md 실전 교훈 §"NO-OP ADR 패턴" 연장** — "인계 항목 실측 재검증" 을 **조사/판정 국면**까지 확장 ("Explore 가 '미결정' 반환 시 debug 실측 선행")
4. **debug-scripts template** — `scripts/_debug-*-tmp.mjs` 패턴 (`.gitignore` 에 이미 등록돼 있을 가능성, 아니면 추가) 문서화

## 관련 노트/링크

- 원 사례 PR: coseo12/astro-simulator#291 (커밋 `d61f657` 최종 수정)
- 원 사례 이슈: coseo12/astro-simulator#292 (drift 조사)
- 관련 volt 이슈:
  - #49 (주석 계약 vs 구현 drift — 버그 생성원)
  - #60 (다운스트림 실측이 최종 가드)
  - #14 (인계 항목 실측 재검증 — NO-OP ADR)
  - #23 (cross-validate 수용 vs 분리 3단 프로토콜) — 본 이슈의 "실측 선행" 제안 수용 시 보강
- 작성 당일 cross-validate 오지시 사례: [이 세션에서 함께 박제된 report]
