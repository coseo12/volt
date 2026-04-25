---
title: monorepo core dist/ stale 빌드 아티팩트로 QA 재검증 false-positive — build→dev 재기동 선행 필수
type: knowledge
source_repo: coseo12/astro-simulator
source_issue: 70
captured_at: 2026-04-25
status: refined
tags: [monorepo, pnpm, build-cache, dev-server, qa, regression, workspace, dist]
related:
  - ./lint-staged-gitignore-silent-revert.md
  - ./manifest-state-recovery-pattern.md
  - ./comment-contract-implementation-drift.md
  - ./state-atomicity-multi-layer-defense.md
---

## 요약

pnpm workspace monorepo 에서 core 패키지(`packages/core`)의 `src/` 수정 후 `pnpm dev` (apps/web) 가 **기존 `dist/` 빌드 아티팩트**를 참조하여 수정이 반영되지 않는다. QA 브라우저 재검증이 이전 실패 수치와 **결정적으로 동일하게 재현**되어 "수정이 효과 없음" 으로 오판하기 쉽다. 해결은 core 빌드 + dev 재기동 선행. CLAUDE.md "빌드 성공 ≠ 동작" / volt #13 "staging 성공 ≠ 커밋 내용" / volt #27 "매니페스트 최신 ≠ 파일 적용 완료" 의 **monorepo 패키지 빌드 캐시 버전**.

## 본문

#### 재현 컨텍스트 (2026-04-23, astro-simulator #313 M2)

- feature/313 브랜치에서 `packages/core/src/scene/tier.ts` + `packages/core/src/scene/solar-system-scene.ts` 수정
- QA sub-agent 가 `pnpm dev` 로 apps/web 실행 후 `agent-browser` 로 브라우저 검증
- 1차 verify 결과 **이전 QA (수정 전 코드) 와 완전히 동일한 실패 수치** (V5 296px / A1 119.9px) 를 3회 결정적으로 재현 → flaky 아님 명확
- QA 가 "수정 내용이 반영되지 않은 상태" 를 의심 → 진단 로그 삽입하여 실제 런타임 코드 추적
- 원인 발견: `packages/core/dist/` 디렉토리가 **수정 전 빌드 아티팩트**를 보유. Next.js / Vite 등 번들러가 core workspace 의 `package.json.main` (dist 경로) 또는 TypeScript path mapping 을 거치며 `dist/` 를 우선 참조
- `pnpm --filter @astro-simulator/core build` 로 dist 리빌드 + dev 서버 재기동 후 **즉시 baseline 복귀** (V5 322px / A1 0.0px)

#### 왜 일어나는가 (메커니즘)

monorepo 에서 core 패키지가 `"main": "dist/index.js"` / `"exports"` 등으로 dist 를 노출하면:

1. 워크스페이스 심볼릭 링크 (`pnpm link`) 는 패키지 루트를 참조
2. 앱(`apps/web`) 의 번들러는 `import { ... } from '@pkg/core'` → 패키지 `main` 경로 → `dist/index.js` 를 읽음
3. dev 모드에서 src 재컴파일 watcher 는 패키지 자체 설정 (`tsc --watch` 등) 에 의존하며, 앱 dev 서버가 자동으로 트리거하지 않음
4. 결과: src 수정 후에도 `dist/` 가 갱신되지 않으면 앱은 이전 코드 실행

TypeScript path mapping (`paths: { "@pkg/core": ["packages/core/src/index.ts"] }`) 로 우회 가능하나, 설정 안 됐거나 번들러가 이를 무시하면 dist 참조 경로로 되돌아간다.

#### 증상 패턴 (진단 체크리스트)

다음이 동시에 관찰되면 dist stale 의심:

- 코드 수정 후 dev 서버 재시작 없이 브라우저 새로고침만 한 경우
- 수정 효과 부재가 **결정적 (매 재현마다 동일 수치)** — flaky 아님
- 빌드/테스트는 pass (vitest 는 `src/` 를 직접 import 하므로 정상)
- `tsc --noEmit` 타입 체크 pass (dist 와 무관)
- CI 는 pass (CI 는 매번 처음부터 빌드)
- 수정된 함수 내부에 `console.log('sanity check')` 삽입해도 로그 부재

#### 방어 루틴

**기본 원칙**: monorepo core 패키지 수정 후 앱 dev 검증 시 다음 중 **하나**를 선택

1. **build 선행**: `pnpm --filter @{scope}/{core-pkg} build` 실행 후 dev 재기동
2. **watch 병행**: `pnpm --filter @{scope}/{core-pkg} build --watch` 별도 터미널에서 상시 실행
3. **TS path mapping**: `tsconfig.json paths` 로 `src/index.ts` 직접 매핑 + 번들러 설정 (next.config.js `transpilePackages` 등) 연동 → dist 우회

**QA 에이전트 체크리스트 가드 (권장)**:

```markdown
## 브라우저 검증 선행 조건 (monorepo 패키지 수정 시)

- [ ] 수정 대상 패키지가 dev 서버 대상 앱의 workspace dependency 인지 확인
- [ ] 수정 대상 패키지의 빌드 아티팩트가 최신인지 확인 (`pnpm --filter <pkg> build`)
- [ ] dev 서버 재기동 (기존 dev 서버 kill 후 재시작) — 포트 재사용 시 stale 프로세스 확인
- [ ] 진단 로그 (`console.log` sanity) 로 런타임 코드가 수정본인지 1회 확인
```

**PR 템플릿 체크박스** (Test plan 섹션에 추가):

```markdown
- [ ] core/shared 패키지 수정 시 dist 빌드 + dev 재기동으로 재현 확인
```

#### 관련 계보 (CLAUDE.md 실전 교훈)

- **빌드 성공 ≠ 동작**: 타입 체크 + 단위 테스트 pass 가 브라우저 동작을 보장하지 않는다 — 본 케이스는 그 **monorepo dist 버전**. 빌드는 pass 인데 런타임은 stale
- **volt #13 staging 성공 ≠ 커밋 내용**: lint-staged 등이 staged 파일 일부를 조용히 유실하는 현상. 본 케이스와 공통: **"반영됐다" 는 표면 신호가 거짓**
- **volt #27 매니페스트 최신 ≠ 파일 적용 완료**: 패키지 매니저 (harness / brew 류) 의 부분 실패. 본 케이스와 공통: **아티팩트와 원본 간 원자성 보장 부재**
- **volt #49 주석 계약 vs 구현 drift**: 본 케이스는 그 역방향 — **구현이 최신이 아니라 아티팩트가 구현보다 뒤짐** (버전 drift)

#### 일반화: 아티팩트 드리프트 3계층 방어 (docs/architecture/state-atomicity-3-layer-defense.md 패턴 재사용)

| 계층 | 방어 | 본 케이스 적용 |
|---|---|---|
| **도중 방어** | 원자적 빌드 — 수정-빌드-재기동이 하나의 트랜잭션 | `pnpm --filter <pkg> build --watch` 상시 실행 |
| **사후 감지** | 아티팩트와 src 의 타임스탬프 / 해시 대조 | `stat -f %m src/index.ts dist/index.js` diff / 번들 해시 비교 |
| **안내** | QA 체크리스트에 검증 루틴 박제 | PR 템플릿 + QA 에이전트 가드 |

본 스킴은 **dev 서버 / 빌드 시스템** 을 추가적 상태 원자성 계층으로 간주. "패키지 워크스페이스는 매니페스트 + 아티팩트 + 앱 런타임의 3자 정합" 이 필요하며 **dev 모드는 3자 정합이 약하다** 는 전제를 채택한다.

## 관련 노트/링크

- astro-simulator PR #315 QA 재검증 코멘트: https://github.com/coseo12/astro-simulator/pull/315#issuecomment-4305500528
- astro-simulator #313 M3 완결: https://github.com/coseo12/astro-simulator/issues/313
- volt #13 (staging 성공 ≠ 커밋 내용): https://github.com/coseo12/volt/issues/13
- volt #27 (매니페스트 최신 ≠ 파일 적용 완료): https://github.com/coseo12/volt/issues/27
- volt #49 (주석 계약 vs 구현 drift): https://github.com/coseo12/volt/issues/49
- CLAUDE.md 실전 교훈 "빌드 성공 ≠ 동작"
