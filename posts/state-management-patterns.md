---
title: 프론트엔드 상태 관리 완전 정복 — Context API부터 Zustand까지 선택 기준 정리
date: 2026-10-11
tags: [상태 관리, React, Zustand, Redux, 프론트엔드]
description: React 애플리케이션에서 상태를 효과적으로 관리하는 다양한 패턴과 라이브러리를 비교하고 상황에 맞는 선택 기준을 제시합니다.
---

## 상태 관리가 왜 어려운가

프론트엔드 개발을 하다 보면 어느 순간 "이 데이터를 어디서 관리해야 하지?" 라는 질문에 맞닥뜨리게 됩니다. 컴포넌트가 늘어나고, 서로 다른 계층에서 같은 데이터를 참조해야 할 때 상태 관리는 빠르게 복잡해집니다. 잘못된 상태 설계는 코드 중복, 예측 불가능한 버그, 불필요한 리렌더링으로 이어집니다.

이 글에서는 React 생태계에서 주로 사용되는 상태 관리 방법들을 비교하고, 프로젝트 규모와 요구사항에 따라 어떤 선택을 해야 하는지 정리합니다.

---

## 상태의 종류를 먼저 파악하자

상태 관리 도구를 선택하기 전에 다루는 상태가 어떤 종류인지 파악하는 것이 중요합니다.

- **로컬 상태**: 특정 컴포넌트 내에서만 필요한 상태 (폼 입력값, 모달 열림 여부 등)
- **공유 상태**: 여러 컴포넌트가 참조하는 상태 (로그인 유저 정보, 장바구니 등)
- **서버 상태**: API로부터 받아오는 비동기 데이터 (목록, 상세 정보 등)
- **URL 상태**: URL에 반영되어야 하는 상태 (페이지 번호, 필터 조건 등)

이 중 서버 상태는 React Query나 SWR 같은 데이터 페칭 라이브러리로 분리하는 것이 요즘 트렌드입니다. 나머지 클라이언트 상태에 집중해서 살펴봅니다.

---

## useState / useReducer — 기본 중의 기본

단일 컴포넌트 내부 상태라면 `useState`로 충분합니다. 상태 업데이트 로직이 복잡해지면 `useReducer`로 전환하면 됩니다.

```tsx
// 간단한 카운터
const [count, setCount] = useState(0);

// 복잡한 폼 상태 관리
const [state, dispatch] = useReducer(formReducer, initialState);
```

**적합한 상황**
- 상태가 해당 컴포넌트 내에서만 사용될 때
- 다른 컴포넌트와 공유할 필요가 없을 때

---

## Context API — 소규모 공유 상태

`useContext`는 prop drilling을 피하기 위한 리액트 내장 솔루션입니다. 단, Context가 변경될 때마다 해당 Context를 구독하는 모든 컴포넌트가 리렌더링된다는 점에 주의해야 합니다.

```tsx
const ThemeContext = createContext<'light' | 'dark'>('light');

function App() {
  const [theme, setTheme] = useState<'light' | 'dark'>('light');
  return (
    <ThemeContext.Provider value={theme}>
      <Layout />
    </ThemeContext.Provider>
  );
}

function Button() {
  const theme = useContext(ThemeContext);
  return <button className={theme}>클릭</button>;
}
```

**적합한 상황**
- 테마, 언어 설정처럼 변경 빈도가 낮은 전역 상태
- 소규모 애플리케이션의 간단한 공유 상태

**주의할 점**
- 자주 변경되는 데이터에 사용하면 성능 문제가 생길 수 있습니다.
- 상태 업데이트 로직이 흩어지기 쉬워 코드 추적이 어려워집니다.

---

## Zustand — 가볍고 실용적인 선택

Zustand는 최근 가장 빠르게 인기를 얻고 있는 상태 관리 라이브러리입니다. 보일러플레이트가 거의 없고, 배우기 쉬우며, 성능도 우수합니다.

```tsx
import { create } from 'zustand';

interface CartStore {
  items: CartItem[];
  addItem: (item: CartItem) => void;
  removeItem: (id: string) => void;
}

const useCartStore = create<CartStore>((set) => ({
  items: [],
  addItem: (item) => set((state) => ({ items: [...state.items, item] })),
  removeItem: (id) =>
    set((state) => ({ items: state.items.filter((i) => i.id !== id) })),
}));

// 컴포넌트에서 사용
function Cart() {
  const items = useCartStore((state) => state.items);
  const removeItem = useCartStore((state) => state.removeItem);
  // ...
}
```

필요한 슬라이스만 구독하므로 불필요한 리렌더링을 방지할 수 있습니다.

**적합한 상황**
- 중소규모 프로젝트의 클라이언트 전역 상태
- Redux가 과하다고 느껴질 때
- 빠른 개발 속도가 필요할 때

---

## Redux Toolkit — 대규모 팀을 위한 표준

Redux는 한때 보일러플레이트가 너무 많다는 비판을 받았지만, Redux Toolkit(RTK)이 등장하면서 크게 개선되었습니다.

```tsx
import { createSlice, configureStore } from '@reduxjs/toolkit';

const counterSlice = createSlice({
  name: 'counter',
  initialState: { value: 0 },
  reducers: {
    increment: (state) => { state.value += 1; },
    decrement: (state) => { state.value -= 1; },
  },
});

export const { increment, decrement } = counterSlice.actions;

const store = configureStore({
  reducer: { counter: counterSlice.reducer },
});
```

**적합한 상황**
- 대규모 팀에서 일관된 패턴이 필요할 때
- 복잡한 비즈니스 로직과 미들웨어가 필요할 때
- Redux DevTools를 활용한 강력한 디버깅이 필요할 때

---

## 선택 기준 정리

| 상황 | 추천 |
|------|------|
| 컴포넌트 내부 상태 | `useState` / `useReducer` |
| 테마·언어 등 정적 전역 설정 | Context API |
| 중소규모 앱의 클라이언트 상태 | Zustand |
| 대규모 팀·복잡한 비즈니스 로직 | Redux Toolkit |
| 서버 비동기 데이터 | React Query / SWR |

---

## 마치며

상태 관리에 정답은 없습니다. 중요한 것은 상태의 종류를 정확히 파악하고, 프로젝트 규모와 팀 상황에 맞는 도구를 선택하는 것입니다. 작은 프로젝트에 Redux를 도입하거나, 반대로 복잡한 대형 서비스에 Context API만으로 버티려 하면 결국 기술 부채가 쌓입니다.

처음에는 `useState`로 시작하고, 필요가 생길 때 Zustand 또는 Redux Toolkit으로 점진적으로 확장하는 전략을 추천합니다. 서버 상태는 처음부터 React Query로 분리하면 훨씬 코드가 깔끔해집니다.
