---
title: develop 의 두 역할 — 통합 스테이징 + PaaS staging env 매핑 (tag trigger 로 대체 불가)
type: knowledge
source_repo: coseo12/harness-setting (v2.15.0 ADR 20260419-gitflow-main-develop 근거 보강)
source_issue: 39
captured_at: 2026-04-19
status: refined
tags: [gitflow, develop, staging, integration-testing, paas, vercel, netlify, deployment]
related:
  - ../meta/patterns/dual-pr-drift-timeline-pattern.md
  - ./release-pr-merge-strategy.md
  - ./gitflow-drift-4tier-classifier.md
---

## 요약

gitflow 에서 `develop` 의 존재 가치를 **배포 격리 (release gate)** 단일 축으로만 평가하면 tag trigger 로 대체 가능한 과설계로 보인다. 하지만 두 번째 축인 **통합 스테이징 (feature 간 상호작용 검증) + PaaS staging environment 매핑** 은 tag trigger 로 **대체 불가능**. 1인 + AI 환경에서도 이 수요는 유효.

## 본문

#### develop 의 두 역할

| 역할 | 설명 | tag trigger 대체 가능? |
|---|---|---|
| 배포 격리 | 언제 배포하냐 통제 | 가능 (`on: push: tags: ['v*']`) |
| **통합 스테이징** | feature 여러 개가 함께 동작하는지 검증 | **불가능** |
| **PaaS staging env 매핑** | main=production / develop=staging / feature=preview | 부분 가능 (도구 의존) |

#### 왜 tag trigger 로 대체 불가능한가

- 각 feature PR 의 preview 는 독립적 (해당 브랜치만 빌드). 다른 feature 가 함께 있지 않음
- main 은 프로덕션 = 바로 배포됨 (자동화 환경)
- feature A 와 B 가 상호작용하는 기능일 때 각자 main 으로 머지 → **프로덕션에서 처음 섞임** → 버그 노출
- develop 은 이 "프로덕션 직전 통합 검증 공간" 역할. tag 트리거는 이 수요와 직교 축

#### PaaS 현실 실측 (2026-04 기준)

| 도구 | 배포 트리거 | tag trigger 네이티브 지원 |
|---|---|---|
| Vercel | Production Branch push | ❌ (Action + CLI 우회만) |
| Netlify | Production branch push | ❌ |
| AWS Amplify | 브랜치 기반 | ❌ |
| Cloudflare Pages | 브랜치 기반 | ❌ |
| Railway / Render | 브랜치 기반 | ❌ |
| GitHub Pages | gh-pages / main push | ❌ |
| npm publish (Actions) | tag 가능 | ✅ |
| Docker/K8s | tag 기반 | ✅ |

웹 앱 PaaS 는 **거의 전부 브랜치 push 기반**. 이런 환경에서 main-only 모델은 "main push = 즉시 production 배포" 가 강제되어 통합 스테이징 공간 부재.

## 교훈

1. **브랜치 모델 결정 시 "배포 격리" 외 축을 잊지 말 것** — 통합 스테이징, staging env 매핑 수요는 tag trigger 로 대체 불가
2. **프로젝트 성격과 배포 도구 현실을 결정 근거에 포함** — 추상적 gitflow vs trunk-based 논의가 아니라 "이 프로젝트가 어떤 PaaS 를 쓰는가" 가 실질 판단 기준
3. **하네스/템플릿 자체 vs 사용자 프로젝트 배포 모델 구분** — 하네스는 tag+gh release 수동, 사용자 프로젝트는 PaaS 자동. gitflow 는 양쪽 공통이고 배포 트리거만 다름
4. **AI 가 초기에 "과설계 의견" 을 강하게 내면 경계** — 맨 처음 Claude 는 "1인 환경에 gitflow 는 과설계" 의견을 냈으나 사용자의 통합 스테이징 수요 지적으로 번복. AI 는 "배포 격리" 축만 본 편향

## 관련 노트/링크

- harness-setting ADR: docs/decisions/20260419-gitflow-main-develop.md (근거 +2 추가)
- harness docs/deployment-patterns.md (신규 가이드)
- harness 릴리스 v2.15.0
