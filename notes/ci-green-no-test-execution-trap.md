---
title: CI 통과 ≠ 테스트 실행 — workflow 언어 감지 템플릿의 no-op 함정
type: knowledge
source_repo: coseo12/harness-setting
source_issue: 48
captured_at: 2026-04-20
status: refined
tags: [ci, github-actions, template, npm-test, audit, observability]
related:
  - ./lint-staged-gitignore-silent-revert.md
  - ./comment-contract-implementation-drift.md
  - ./flaky-test-suppress-vs-root-cause.md
---

## 요약

"언어 자동 감지" 범용 CI 템플릿이 `echo` 만 수행하고 실제 `npm test` / `pytest` 등을 돌리지 않는 경우가 있다. 최근 PR 들이 **초록 체크로 머지되지만 실제로는 테스트가 돌지 않은 상태**. PR 자동 체크가 PASS 한다는 표면 신호만 보지 말고 **Actions 로그의 테스트 출력 존재 여부** 를 정기 감사한다.

## 본문

#### 관찰 사례

harness-setting `.github/workflows/ci.yml` 의 `detect-and-test` 잡:

```yaml
- name: Node.js 테스트
  if: hashFiles('package.json') != ''
  run: |
    node_version=$(jq -r '.engines.node // "20"' package.json | grep -oE '[0-9]+' | head -1)
    echo "Node.js ${node_version} 사용"
  # Node 설정은 프로젝트별로 추가
```

- Python/Go/Rust 브랜치도 동일하게 `echo` 만 수행
- 각 프로젝트가 자기 블록을 커스터마이즈하라는 의도였으나 harness 저장소 자체에서는 후속 커스터마이즈가 누락
- 결과: 최근 세션 4개 PR (#144, #147, #150, #154) 이 CI 에서 `npm test` 가 돌지 않은 채 머지됨. 로컬 `npm test` 통과만 신뢰 기반

#### 진단 신호

1. **실행 시간**: Node 테스트가 포함된 CI 가 5초 안에 PASS → 의심. `npm ci` 만으로도 수십 초 소요되므로 테스트 실행 없음 의심
2. **Actions 로그**: step log 에 실제 테스트 러너 출력 (`ℹ tests N / ℹ pass N`, `PASSED`, `ok N`) 부재
3. **CI 구조**: `detect-and-test` 같은 범용 템플릿 이름 + step 내용이 `echo` 뿐

#### 범용화

- **CI 감사 루틴**: 리포지토리 초기 설정 후 최소 1회, 이후 주기적 (분기 1회) 으로 CI Actions 로그에서 실제 테스트 출력이 있는지 확인
- **고의적 실패 PR 실측**: 테스트 1건 일부러 깨뜨린 draft PR 로 CI 가 실제로 red 로 전환되는지 체크 후 revert
- **"로컬 통과 = 안전" 가정 금지**: CI 가 실질 회귀 게이트로 동작하지 않으면 로컬 miss 가 곧 main 오염
- **템플릿 유산 패턴 주의**: "범용 언어 감지" 형태의 CI 템플릿은 커스터마이즈 누락 시 no-op 로 남기 쉽다

#### 인접 원칙

- CLAUDE.md `실전 교훈` 의 **"빌드 성공 ≠ 동작하는 앱"** 확장 — CI 통과도 같은 계열의 표면 신호
- **"staging 성공 ≠ 커밋 내용"** (volt [#13](https://github.com/coseo12/volt/issues/13)) 과 연결 — 파이프라인 각 단계 신호가 실제 결과를 증명하지 않을 수 있다
