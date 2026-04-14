---
title: 차분 구독(Zustand subscribe next/prev)으로 UI → Scene 전파
type: pattern
source_repo: coseo12/astro-simulator
source_issue: 4
captured_at: 2026-04-14
status: inbox
tags: [zustand, react, scene-sync, babylonjs]
related: []
---

## 배경/상황

Zustand store의 여러 필드(엔진 종류, 질량 배수 맵 등)가 변경될 때 Babylon 씬의 해당 API(`setPhysicsEngine`, `setBodyMassMultiplier`)로 전파해야 했다. useEffect로 각 필드마다 훅을 만드는 대신, `store.subscribe((next, prev) => ...)` 한 곳에서 차분을 감지해 처리하는 패턴이 안정적이었음.

## 내용

```ts
// sim-canvas.tsx 패턴
const unsubEngine = useSimStore.subscribe((state, prev) => {
  if (state.physicsEngine !== prev.physicsEngine) {
    solar.setPhysicsEngine(state.physicsEngine);
  }
  if (state.massMultipliers !== prev.massMultipliers) {
    // 차분 diff — 삭제된 키는 1.0으로 복원
    const prevKeys = new Set(Object.keys(prev.massMultipliers));
    const nextKeys = new Set(Object.keys(state.massMultipliers));
    for (const k of prevKeys) {
      if (!nextKeys.has(k)) solar.setBodyMassMultiplier(k, 1);
    }
    for (const [k, v] of Object.entries(state.massMultipliers)) {
      if (prev.massMultipliers[k] !== v) solar.setBodyMassMultiplier(k, v);
    }
  }
});
// cleanup: unsubEngine();
```

장점:
- React 컴포넌트 외부(useEffect 의존성 배열 관리 불필요) 동작.
- 여러 필드 구독을 한 콜백에 묶어 순서 제어 가능.
- Map/Record 차분에서 "제거된 키" 복원 로직이 명확.
- 언마운트 시 단일 unsub 함수로 정리 — 누수 위험 낮음.

주의:
- callback이 state·prev 레퍼런스 동일성에만 의존 → store 측에서 불변 업데이트 보장 필요.
- 초기 상태는 subscribe가 호출하지 않으므로 생성 시 1회 initial sync 필요(`useSimStore.getState()`).

## 교훈 / 다음에 적용할 점

- **React 외부의 스테이트풀 객체(캔버스·엔진)와 store 동기화는 useEffect보다 `store.subscribe` 단일 진입점이 관리 용이.**
- Map/Record 전파는 삭제 복원을 명시적으로 다뤄야 함 — 기본값/초기값 정의 필요.
- skill 또는 가이드에 "React 외부 시스템과 Zustand 동기화" 패턴 문서화 가치.
