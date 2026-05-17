---
title: Strict Assertion vs Fallback 자기모순 — create-pr drift 가드 fail-fast 설계
type: knowledge
source_repo: coseo12/astro-simulator
source_issue: 107
captured_at: 2026-05-17
status: inbox
tags: [strict-assertion, fallback, self-contradiction, drift-guard, fail-fast, design-tension]
related:
  - ../notes/comment-contract-implementation-drift.md
  - ../notes/cross-validate-dod-self-contradiction.md
  - ../notes/cross-validate-plan-mode-bypass-guard.md
---

## 요약

PR 본문 7 체크박스 base 를 `.github/PULL_REQUEST_TEMPLATE.md` 에서 동적 읽기로 보장하려는 가드 (`create-pr` 스킬) 가 fallback 분기를 가지면 strict assertion 의미가 무력화되는 자기모순. **fail-fast 만 박제** + fallback 제거가 정합. drift 가드 일반화 설계 원칙.

## 본문

**문제 (PR #478 #471 create-pr Strict Assertion 도입 시점)**:

초안 (fallback 박제):
```javascript
async function readPRTemplate() {
  try {
    return await readFile('.github/PULL_REQUEST_TEMPLATE.md', 'utf-8');
  } catch (err) {
    console.warn('PR template not found, using fallback base');
    return DEFAULT_TEMPLATE_BASE;  // ← fallback 분기
  }
}
```

자기모순:
- strict assertion 의도: "PR 본문 7 체크박스가 PULL_REQUEST_TEMPLATE.md 와 정확 일치하는지 검증"
- fallback 분기: 템플릿 파일 부재/오류 시 `DEFAULT_TEMPLATE_BASE` 로 대체
- → 파일 부재가 가장 위험한 drift 인데 fallback 이 그것을 silent skip ⇒ 가드 무력화

**해결 (PR #478 최종)**:

fail-fast 만:
```javascript
async function readPRTemplate() {
  const content = await readFile('.github/PULL_REQUEST_TEMPLATE.md', 'utf-8');
  // 부재/오류 시 ENOENT 등 native error 가 그대로 throw → CI fail
  return content;
}
```

**원칙 일반화** — drift 가드 설계 시:
- **fallback = silent skip = 가드 무력화** ⇒ 절대 박제 금지
- 가드는 본질적으로 negative test 성격 → "정상 = pass, 비정상 = fail" 의 이분법
- "비정상이지만 fallback 으로 동작" 박제는 **strict assertion 의미 자체를 폐기**
- 운영 비용 (파일 부재 시 즉시 CI fail) 은 운영 측 책임 — 가드 측이 흡수하면 안 됨

**관련 패턴**:
- harness real-lessons "주석 계약 vs 구현 drift" (volt #49) — fallback 이 silent 흡수하는 변형
- "fail loud, fail early" (defensive programming 원칙) 의 가드 측 적용

**구분 — strict assertion vs validation**:
- strict assertion: 가드 측 (drift 감지) — fail-fast 의무
- validation: 사용자 입력 측 — graceful degradation 합리적
- 둘을 혼동하면 가드가 validation 흉내 내며 silent skip

## 관련 노트/링크

- 사례: astro-simulator PR #478 (#471 create-pr Strict Assertion, 2026-05-15)
- 관련 volt: #49 (주석 계약 vs 구현 drift), #66 (cross-validate self-contradiction), #99 (cross-validate plan-mode 우회)
