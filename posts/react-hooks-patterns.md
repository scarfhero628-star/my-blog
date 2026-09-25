---
title: React 훅을 제대로 활용하는 법 — 실전에서 바로 쓰는 커스텀 훅 패턴
date: 2026-09-25
tags: [React, JavaScript, 커스텀 훅, 웹 개발, 프론트엔드]
description: React의 기본 훅부터 커스텀 훅 설계 원칙까지, 실무에서 코드 재사용성과 가독성을 높이는 훅 패턴을 소개합니다.
---

React 16.8에서 훅(Hooks)이 도입된 이후 컴포넌트 작성 방식이 완전히 바뀌었습니다. 클래스 컴포넌트 없이도 상태와 사이드 이펙트를 다룰 수 있게 되었고, 무엇보다 로직을 재사용하기가 훨씬 쉬워졌습니다. 하지만 훅을 잘 쓰는 것과 단순히 쓰는 것은 다릅니다. 이 글에서는 기본 훅을 올바르게 사용하는 법부터, 실무에서 진가를 발휘하는 커스텀 훅 패턴까지 단계별로 살펴봅니다.

---

## 기본 훅 제대로 이해하기

### useState: 상태는 최소화하라

컴포넌트 상태를 무분별하게 늘리면 렌더링 비용이 커지고 코드 추적이 어려워집니다.

```jsx
// 나쁜 예: 파생 가능한 값을 별도 상태로 관리
const [firstName, setFirstName] = useState('');
const [lastName, setLastName] = useState('');
const [fullName, setFullName] = useState('');

// 좋은 예: 파생 값은 계산으로 처리
const [firstName, setFirstName] = useState('');
const [lastName, setLastName] = useState('');
const fullName = `${firstName} ${lastName}`.trim();
```

**규칙:** 다른 상태에서 계산할 수 있는 값은 state로 관리하지 않는다.

### useEffect: 의존성 배열을 정직하게 쓰라

`useEffect`의 의존성 배열에서 값을 빼먹으면 클로저 문제로 인해 예상치 못한 버그가 생깁니다. ESLint의 `exhaustive-deps` 규칙을 켜두고 경고를 무시하지 마세요.

```jsx
// 나쁜 예: count가 의존성에 없어서 항상 초기값 0을 참조
useEffect(() => {
  const id = setInterval(() => {
    setCount(count + 1); // count는 항상 0
  }, 1000);
  return () => clearInterval(id);
}, []); // 의존성 누락

// 좋은 예: 함수형 업데이트로 의존성 제거
useEffect(() => {
  const id = setInterval(() => {
    setCount(prev => prev + 1); // 항상 최신값 기반
  }, 1000);
  return () => clearInterval(id);
}, []);
```

### useMemo와 useCallback: 꼭 필요할 때만 쓰라

메모이제이션은 공짜가 아닙니다. 비교 연산 비용이 발생하므로, 실제로 비싼 연산이나 참조 동일성이 중요한 경우에만 사용하세요.

```jsx
// 불필요한 useMemo (단순 연산)
const doubled = useMemo(() => count * 2, [count]); // 오버킬

// 적절한 useMemo (무거운 필터링 연산)
const filteredList = useMemo(
  () => largeList.filter(item => item.active && item.score > threshold),
  [largeList, threshold]
);
```

---

## 커스텀 훅: 로직을 분리하는 핵심 도구

커스텀 훅은 단순히 훅을 감싸는 것이 아닙니다. **컴포넌트에서 복잡한 로직을 분리해 재사용 가능하게 만드는 패턴**입니다.

### 패턴 1: 데이터 페칭 훅

API 호출 로직을 매번 컴포넌트에 작성하면 `loading`, `error`, `data` 상태 관리가 반복됩니다. 커스텀 훅으로 추상화하세요.

```jsx
function useFetch(url) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    let cancelled = false;

    async function fetchData() {
      try {
        setLoading(true);
        const response = await fetch(url);
        if (!response.ok) throw new Error('Network error');
        const json = await response.json();
        if (!cancelled) setData(json);
      } catch (err) {
        if (!cancelled) setError(err.message);
      } finally {
        if (!cancelled) setLoading(false);
      }
    }

    fetchData();
    return () => { cancelled = true; };
  }, [url]);

  return { data, loading, error };
}

// 사용
function UserProfile({ userId }) {
  const { data: user, loading, error } = useFetch(`/api/users/${userId}`);
  if (loading) return <Spinner />;
  if (error) return <ErrorMessage message={error} />;
  return <div>{user.name}</div>;
}
```

`cancelled` 플래그로 컴포넌트 언마운트 후 상태 업데이트를 막는 것이 중요합니다.

### 패턴 2: 로컬 스토리지 동기화 훅

상태를 브라우저 스토리지와 자동으로 동기화하는 훅입니다.

```jsx
function useLocalStorage(key, initialValue) {
  const [value, setValue] = useState(() => {
    try {
      const stored = window.localStorage.getItem(key);
      return stored ? JSON.parse(stored) : initialValue;
    } catch {
      return initialValue;
    }
  });

  const setStoredValue = useCallback((newValue) => {
    try {
      setValue(newValue);
      window.localStorage.setItem(key, JSON.stringify(newValue));
    } catch (error) {
      console.error('LocalStorage error:', error);
    }
  }, [key]);

  return [value, setStoredValue];
}

// 사용: useState와 동일한 API
const [theme, setTheme] = useLocalStorage('theme', 'light');
```

초기화를 지연 함수로 처리해 불필요한 `localStorage` 접근을 막습니다.

### 패턴 3: 디바운스 훅

검색 입력 처리에 자주 쓰이는 디바운스 로직입니다.

```jsx
function useDebounce(value, delay = 500) {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => setDebouncedValue(value), delay);
    return () => clearTimeout(timer);
  }, [value, delay]);

  return debouncedValue;
}

// 사용
function SearchBox() {
  const [query, setQuery] = useState('');
  const debouncedQuery = useDebounce(query, 300);

  useEffect(() => {
    if (debouncedQuery) searchApi(debouncedQuery);
  }, [debouncedQuery]);

  return <input value={query} onChange={e => setQuery(e.target.value)} />;
}
```

---

## 커스텀 훅 설계 원칙

좋은 커스텀 훅을 만들기 위한 실전 원칙입니다.

1. **이름은 항상 `use`로 시작하라** — React가 훅 규칙을 적용하려면 반드시 필요합니다.
2. **한 가지 일만 하라** — `useUserProfile`은 유저 정보만, `useModal`은 모달 상태만 담당해야 합니다.
3. **내부 구현을 숨기고 필요한 것만 반환하라** — 불필요한 상태나 함수를 외부에 노출하지 마세요.
4. **의존성을 외부에서 주입하라** — URL, 설정 값 등은 하드코딩하지 말고 매개변수로 받으세요.
5. **정리(cleanup) 로직을 빠뜨리지 마라** — 이벤트 리스너, 타이머, 구독은 반드시 해제해야 합니다.

---

## 마치며

커스텀 훅은 React의 가장 강력한 패턴 중 하나입니다. 복잡한 컴포넌트를 마주쳤을 때 "이 로직을 훅으로 분리할 수 있을까?"라고 먼저 생각하는 습관을 들이세요. 처음에는 간단한 `useFetch`나 `useDebounce`부터 시작해 점점 프로젝트에 맞는 훅 라이브러리를 쌓아가면, 코드 품질과 생산성이 눈에 띄게 달라질 것입니다.
