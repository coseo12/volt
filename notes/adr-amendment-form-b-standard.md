---
title: ADR Amendment 형식 — B 형식 표준화 (현존 문서 보존 + 신설 §Amendment 섹션)
type: knowledge
source_repo: coseo12/astro-simulator
source_issue: 104
captured_at: 2026-05-17
status: refined
tags: [adr, amendment, document-pattern, ssot, decision-record]
related:
  - ./adr-amendment-3way-markup-ssot.md
  - ./adr-amendment-location-decision-tree.md
---

## 요약

ADR 수정 시 본문 mutation 두 형식 검토 후 **B 형식 (현존 문서 보존 + §Amendment N 섹션 신설)** 을 표준 채택. A 형식 (본문 직접 수정 + strikethrough) 은 변경 이력 가독성 + diff 신뢰성 측면에서 열등. volt #91 (ADR Amendment 위치 결정 트리) 의 형식 측면 보강.

## 본문

**ADR Amendment 두 형식**:

**A 형식 — 본문 직접 mutation**:
```markdown
## 결정

~~원안~~ → **Amendment 1 (2026-05-16): 변경된 결정**

(strikethrough + footnote 또는 inline 변경 노트)
```

장점: 단일 SSoT 유지. 단점:
- diff 가 mutation 으로 표현되어 변경 이력 불명확
- strikethrough 가 많아지면 가독성 급락
- 마크다운 렌더링이 일관되지 않음 (GitHub vs VS Code vs 일반 viewer)

**B 형식 — 신설 §Amendment 섹션** (표준 채택):
```markdown
## 배경
(원본 보존)

## 결정
(원본 보존)

...

## Amendment 1 (2026-05-16)

### 배경
(왜 수정 — 이슈 #N, 트리거 조건 등)

### 변경 사항
- §재검토 조건 #5 → "Phase 2 N=3 OR 30일 트리거" 추가
- §측정 지표 → health metric 박제

### 영향 범위
(downstream 영향)
```

장점:
- 원본 + 변경 history append-only 로 차곡차곡 누적
- diff 가 깔끔 (신설 섹션만 +)
- "원본 의도 vs 현재 결정" 비교 용이
- 후속 Amendment N+1, N+2 가 자연 append

**ADR 운영 컨벤션 (B 형식 기준)**:
1. 같은 이슈가 ADR 결정을 수정하면 **mutation** (A → B 의 B 형식 §Amendment 추가)
2. 다른 이슈가 ADR 결정을 수정하면 **신규 ADR + cross-link** (`Supersedes: 20260515-...`)
3. 판단 기준: volt #91 "ADR Amendment 위치 결정 트리"

**첫 사례** (astro-simulator):
- ADR `20260515-harness-managed-divergent-pattern.md` (Z 패턴)
- Amendment 1 (PR #486, 2026-05-16): Phase 2 N=3 OR 30일 트리거 + health metric
- Amendment 2 (PR #489): N=10 OR 90일 완화 (옵션 C)
- Amendment 3 (PR #490): 자동 탐지 workflow 박제

**적용 조건**:
- ADR 결정 자체 수정 시 (배경 정정은 본문 직접 수정 가능)
- 후속 이슈가 추가 결정 조항을 도출했을 때
- §재검토 조건 발화 후 사용자 결정 박제 시

## 관련 노트/링크

- 사례: astro-simulator ADR `20260515-harness-managed-divergent-pattern.md` (Amendment 1/2/3)
- 관련 volt: #83 (ADR 정정 패턴 SSoT — strikethrough + footnote + Amendment 3중 마크업), #91 (Amendment 위치 결정 트리)
