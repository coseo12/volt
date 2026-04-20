---
title: test flaky 대응 — concurrency=1 누르기는 증상 마스킹, 로그 기반 진단 우선
type: knowledge
source_repo: coseo12/harness-setting
source_issue: 50
captured_at: 2026-04-20
status: refined
tags: [testing, flaky, concurrency, debugging, diagnostic-routine, root-cause]
related:
  - ./ci-green-no-test-execution-trap.md
  - ./comment-contract-implementation-drift.md
---

## 요약

병렬 테스트 flaky 발견 시 `--test-concurrency=1` 또는 `--jobs 1` 로 누르는 것은 **증상 마스킹**. 근본 원인은 보통 공유 리소스 / 카테고리 오분류 / I/O race 이고, 임시 조치는 실행 시간 비용 + 근본 원인 은폐 두 단점. **실패 시 stderr 출력에서 영향받은 객체를 특정** → 원인 추적 루트로 바로 연결된다.

## 본문

#### 관찰 사례

- v2.24.0 (#153) 착수 중 `test/previous-sha256.test.js:140` / `test/update-verification.test.js:172` 가 병렬 실행 시 약 75% 빈도로 flaky 판정
- 임시 조치: `scripts.test` 를 `node --test --test-concurrency=1` 로 변경 → 안정 확보, 실행 시간 6s → 18s (**3배 증가**)
- "우선순위 low" 로 후속 이슈 #157 분리. Not-Planned close 제안까지 나왔으나 **D2 (깊은 분석)** 선택
- 실패 시 stderr 출력: `harness update: post-apply 검증 실패 N건 — .claude/logs/cross-validate-structure-20260420-141012.log, docs/agents-guide.md`
- `.claude/logs/cross-validate-*.log` 가 rolledBack 대상이라는 단서로 카테고리 오분류 식별 → `categorize('.claude/logs/...')` 반환 `'atomic'` 확인 → 주석 계약 대조 → 누락 규칙 1 라인 추가로 근본 해소

#### 누르기 임시 조치의 비용

| 축 | `--test-concurrency=1` 적용 | 근본 해소 후 |
|---|---|---|
| 실행 시간 | 18s (로컬) / 15s (CI) | 7.5s (로컬) / 2.4s (CI) |
| 테스트 격리 강도 | 높음 (순차) | 낮음 (병렬 재도입) |
| 근본 원인 은폐 | ⚠️ 예 | 해소 |
| 다른 flaky 감지 능력 | 저하 (순차가 다른 concurrency bug 도 은폐) | 유지 |

#### 범용 진단 루트 (재사용)

1. **병렬 반복 실행 + stderr / 실패 assert 메시지 수집** — 10~20회 돌려서 실패율 + 실패 시 출력 패턴 기록
2. **실패 시 영향받은 객체 특정** — stderr/assert 에서 파일명/키/리소스 이름 추출
3. **이름 패턴 분석** — 공통 접두사 / 디렉토리 / 카테고리 (예: "실패 파일이 전부 `.claude/logs/` 하위")
4. **해당 객체의 분기 / 카테고리 결과 확인** — categorize 류 함수, configuration lookup
5. **주석 계약 / 문서 규칙 대조** — 주석-구현 drift 발견 (#157 케이스)
6. **수정 + 회귀 가드** — 8회 이상 연속 병렬 통과 실측 (1-2회 통과 = 증명 부족)

#### 안티패턴

- **"flaky = concurrency 감축"** 으로 직행 — 항상 의심부터 하고 stderr 확인
- **"flaky = retry"** — CI 재시도 플래그 추가. 숨은 비용: CI 시간 증가 + 진짜 실패 탐지 지연
- **"flaky = skip/xfail"** — 근본 원인을 영구 은폐
- **"flaky = low 우선순위 방치"** — #157 도 처음엔 low 로 close 제안됐으나 D2 로 뒤집어 1 라인 수정 + 60~80% CI 단축의 큰 이득 확보

#### 수용 기준 (근본 해소 실증)

- **8회 이상 연속 병렬 실행 0 실패** (timing 문제의 불확실성 고려 시 충분한 샘플)
- **임시 조치 제거 상태에서 통과** — concurrency=1, retry 등 완화 수단 없이
- **실행 시간 회복** — 누르기 전 수준 (또는 더 빠르게)

#### 인접 원칙

- CLAUDE.md `실전 교훈` "빌드 성공 ≠ 동작" 의 테스트 영역 확장 — "테스트 통과 (concurrency=1 로) ≠ 버그 없음"
- CLAUDE.md "수치 DoD 미달 시 측정 방법 검증 우선" 과 연결 — flaky 발견 시 "측정 방법 (테스트 구조) 자체를 재평가" 라기보다 "측정 대상 (프로덕션 코드) 의 숨은 버그" 일 가능성 우선 검토
