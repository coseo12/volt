---
title: 메모리 keyword-only 박제 함정 — 메모리에 키워드만 박제되고 상세 사례 미박제 시 재발견 비용 폭증
type: report
source_repo: coseo12/astro-simulator
source_issue: 113
captured_at: 2026-05-25
status: inbox
tags: [memory-anti-pattern, keyword-only-capture, context-loss, knowledge-decay, capture-volt-quality]
related:
  - ../notes/meta-risk-bypass-2stage-zero-fire.md
---

## 배경/상황

Claude Code 자동 메모리 시스템 (`MEMORY.md` 및 산하 `.md` 파일) 에 박제할 때 **재사용 가능한 패턴 이름 (키워드) 만 박제하고 상세 사례·근거·재현 절차를 박제하지 않으면**, 후속 세션에서 해당 패턴을 회수할 때 컨텍스트 부재로 재박제 불가. 본 회고는 그 함정을 자기 박제하는 메타 retrospective.

## 내용

**관찰 사례 (2026-05-17 본 세션)**:

메모리 누적 25+건 일괄 volt 캡처 작업 중 3건 (Brand Drift 처리 / chicken-and-egg dogfood 옵션 b / 메타 위험 우회 패턴) 의 메모리 박제가 **키워드만 있고 상세 사례 부재**:

- `Brand Drift 처리 패턴` — astro-simulator 프로젝트의 어떤 사례인지 불명. UI 브랜드 일관성 drift? 기능명 drift? 사례 컨텍스트 0
- `chicken-and-egg dogfood 옵션 b` — volt #78 (옵션 a: PR 분리) 의 대안 변형이라는 키워드만 박제. 옵션 b 가 무엇인지 (단일 PR 내 step 비활성화? 임시 baseline? fallback 분기?) 명시 부재
- `메타 위험 우회 패턴` — "architect + dev D4 2회 연속" 키워드. 다행히 PR #482 #479 사례로 추정 박제 가능 (volt #112 박제)

3건 중 1건 (메타 위험 우회) 만 추정 박제 가능, 2건은 메타 함정으로 묶어 별도 박제.

**근본 원인 (메모리 박제 시점의 인지 편향)**:
- 박제 시점에서는 패턴 이름만 보아도 사례가 떠오름 → 키워드만 박제해도 충분하다고 착각
- 1~2주 후 세션 컨텍스트 reset 시 키워드만으로는 사례 회수 불가
- 메모리 박제의 **시간 지연 비용** 을 박제 시점에서 인지 못 함

**예방 (메모리 박제 시점 의무)**:
1. **패턴 이름 + 사례 1줄** 박제 의무
   - 좋은 예: `Brand Drift 처리 — astro-simulator #X PR Y 에서 색상 코드 drift 발견 → 디자인 토큰 SSoT 박제로 해결`
   - 나쁜 예: `Brand Drift 처리 패턴`
2. **사례 박스 (1~3줄)** 박제 의무 — 후속 세션 회수 시 최소 컨텍스트
3. **관련 이슈/PR 번호** 박제 의무 — `#479` / `PR #482` 등 — 후속 회수 시 즉시 본문 조회 가능

**메타 — 본 retrospective 자체가 메타 함정**:
- 메모리 박제 시점에서 "왜 이 패턴이 중요한지" 의 동기를 함께 박제해야 미래 회수 가능
- 본 회고도 "동기" 명시 박제 의무 — "메모리 박제는 미래 자신에게 보내는 편지" 비유
- 키워드만 박제는 "오늘의 나" 에게는 충분하나 "내일의 나" 에게는 컨텍스트 0

**capture-volt 스킬 측 개선 권고**:
- capture-volt 호출 시 메모리 박제 키워드 → volt 상세 박제 변환 도우미 추가
- 박제 직전 "사례·근거·재현 절차" 3 필드 의무화
- 키워드만 박제 거부 (필드 부족 시 fail)

## 교훈 / 다음에 적용할 점

- **메모리 박제 의무 — 키워드 + 사례 1줄 + 이슈/PR 번호 3 set**
- **메모리 회수 의무 — 키워드만 발견 시 사용자에게 컨텍스트 요청 또는 메타 함정 박제**
- harness CLAUDE.md "auto memory" 섹션에 본 함정 박제 후보
- 본 retrospective 자체가 "메모리 박제 품질" 회로

## 관련 노트/링크

- 본 세션 사례: 2026-05-17 volt 17건 캡처 작업 중 3건 keyword-only 박제 발견
- 추정 박제 성공: volt #112 (메타 위험 우회 — PR #482 컨텍스트 회수 성공)
- 추정 박제 무산: Brand Drift 처리 / chicken-and-egg dogfood 옵션 b (컨텍스트 0)
- 관련 volt: 모든 자동 메모리 운영 회고
