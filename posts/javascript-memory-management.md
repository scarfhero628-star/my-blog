---
title: JavaScript 메모리 관리 완전 가이드 — 메모리 누수를 예방하고 성능을 높이는 방법
date: 2026-10-31
tags: [JavaScript, 메모리 관리, 성능, 웹 개발, 메모리 누수]
description: JavaScript에서 흔히 발생하는 메모리 누수의 원인을 파악하고, 가비지 컬렉션 원리를 이해해 성능 문제를 예방하는 실전 방법을 소개합니다.
---

## 왜 JavaScript 메모리 관리가 중요한가

JavaScript는 가비지 컬렉터(Garbage Collector, GC)가 메모리를 자동으로 관리해 주기 때문에 개발자가 직접 메모리를 해제할 필요가 없습니다. 하지만 "자동 관리"라는 말이 "메모리 걱정 없음"을 의미하지는 않습니다. 잘못 작성된 코드는 GC가 해제하지 못하는 객체를 계속 쌓아 메모리 누수(Memory Leak)를 유발하고, 결국 앱이 느려지거나 브라우저 탭이 강제 종료되는 상황을 만들어 냅니다.

특히 SPA(Single Page Application) 환경에서는 페이지를 새로 고침하지 않고 장시간 사용하기 때문에, 누수가 누적될수록 성능 저하가 더욱 두드러집니다. 이 글에서는 가비지 컬렉션의 원리부터 실제 누수 패턴과 해결책까지 단계별로 정리합니다.

## 가비지 컬렉션 동작 원리

### 도달 가능성(Reachability) 개념

JavaScript의 GC는 **도달 가능한(reachable) 객체**를 살리고, 그렇지 않은 객체를 메모리에서 제거합니다. 도달 가능한 객체란 루트(전역 변수, 현재 실행 중인 함수의 로컬 변수 등)에서 참조 체인을 따라 접근할 수 있는 객체를 말합니다.

```js
let user = { name: 'Alice' }; // user → { name: 'Alice' } 참조
user = null; // 참조가 끊겼으므로 GC 대상
```

### Mark-and-Sweep 알고리즘

현대 JavaScript 엔진은 **Mark-and-Sweep** 알고리즘을 사용합니다.

1. **Mark 단계**: 루트에서 시작해 도달 가능한 모든 객체를 표시(mark)합니다.
2. **Sweep 단계**: 표시되지 않은 객체를 메모리에서 제거합니다.

이 과정 덕분에 **순환 참조**도 문제없이 처리됩니다.

```js
// 순환 참조 — 옛날 참조 카운팅 방식에서는 누수였지만 Mark-and-Sweep은 해제 가능
function createCycle() {
  const a = {};
  const b = {};
  a.ref = b;
  b.ref = a;
  // 함수 종료 후 a, b 모두 루트에서 도달 불가 → GC 수거
}
```

## 흔한 메모리 누수 패턴

### 1. 전역 변수의 남용

`var` 또는 선언 없이 할당한 변수는 전역 객체(`window`)에 붙어 앱이 살아있는 동안 메모리에 남습니다.

```js
// 나쁜 예
function processData(data) {
  result = data.map(/* ... */); // 전역 변수 생성
}

// 좋은 예
function processData(data) {
  const result = data.map(/* ... */);
  return result;
}
```

### 2. 이벤트 리스너 미제거

DOM 요소나 이벤트 에미터에 리스너를 추가하고 제거하지 않으면, 리스너가 참조하는 클로저와 관련 객체가 계속 살아 있습니다.

```js
// 나쁜 예 — 컴포넌트를 제거해도 리스너는 남아있음
window.addEventListener('resize', handleResize);

// 좋은 예 — 사용 후 반드시 제거
window.addEventListener('resize', handleResize);
window.removeEventListener('resize', handleResize);
```

React를 사용한다면 `useEffect`의 클린업 함수를 활용하세요.

```js
useEffect(() => {
  window.addEventListener('resize', handleResize);
  return () => window.removeEventListener('resize', handleResize);
}, []);
```

### 3. 클로저로 인한 대용량 데이터 참조

클로저가 불필요하게 큰 객체를 참조하면 해당 객체는 클로저가 살아있는 한 메모리에 유지됩니다.

```js
// 나쁜 예 — largeData 전체가 메모리에 유지됨
function createHandler(largeData) {
  return function() {
    return largeData.id; // id만 필요하지만 largeData 전체 참조
  };
}

// 좋은 예 — 필요한 값만 추출
function createHandler(largeData) {
  const { id } = largeData;
  return function() {
    return id;
  };
}
```

### 4. 삭제되지 않는 타이머와 인터벌

`setInterval`은 명시적으로 `clearInterval`을 호출하기 전까지 계속 실행되며, 콜백 내부에서 참조하는 객체도 GC 대상이 되지 않습니다.

```js
// 나쁜 예
const timer = setInterval(() => {
  updateDashboard(bigDataset);
}, 1000);

// 좋은 예 — 컴포넌트 언마운트 등 적절한 시점에 제거
const timer = setInterval(() => {
  updateDashboard(bigDataset);
}, 1000);

clearInterval(timer); // 사용 완료 후 반드시 제거
```

### 5. WeakMap·WeakSet을 활용한 캐시

일반 `Map`이나 `Set`을 캐시로 사용할 때 키 객체가 더 이상 필요 없어도 `Map`이 강한 참조를 유지하므로 GC가 수거하지 못합니다. `WeakMap`을 사용하면 키 객체가 다른 곳에서 참조되지 않을 때 자동으로 수거됩니다.

```js
// 일반 Map — 키가 삭제되어도 Map이 참조를 유지
const cache = new Map();
cache.set(domElement, computedData);

// WeakMap — domElement가 DOM에서 제거되면 자동으로 GC 수거
const weakCache = new WeakMap();
weakCache.set(domElement, computedData);
```

## Chrome DevTools로 메모리 누수 찾기

### Memory 탭 활용

1. DevTools를 열고 **Memory** 탭으로 이동합니다.
2. **Heap snapshot**을 찍어 현재 메모리 상태를 확인합니다.
3. 누수가 의심되는 동작을 수행한 후 다시 스냅샷을 찍습니다.
4. **Comparison** 뷰에서 두 스냅샷의 차이를 비교해 살아남은 객체를 확인합니다.

### Performance 탭 활용

**Performance** 탭의 **Record** 기능으로 시간에 따른 힙 사용량 변화를 기록하세요. 힙이 계속 우상향 그래프를 그린다면 누수를 의심해야 합니다.

## WeakRef와 FinalizationRegistry 활용 (ES2021+)

ES2021부터는 `WeakRef`와 `FinalizationRegistry`를 사용해 GC와 더 세밀하게 상호작용할 수 있습니다.

```js
// WeakRef — GC가 수거할 수 있는 약한 참조
let obj = { data: '중요한 데이터' };
const weakRef = new WeakRef(obj);

// 나중에 참조가 아직 살아있는지 확인
const derefObj = weakRef.deref();
if (derefObj) {
  console.log(derefObj.data);
}

// FinalizationRegistry — 객체가 수거될 때 콜백 실행
const registry = new FinalizationRegistry((heldValue) => {
  console.log(`${heldValue} 객체가 GC에 의해 수거되었습니다`);
});

registry.register(obj, '추적 대상');
```

## 핵심 정리

| 누수 패턴 | 문제 원인 | 해결책 |
|-----------|-----------|--------|
| 전역 변수 | 앱 종료 전까지 메모리 점유 | `const`/`let`으로 스코프 제한 |
| 이벤트 리스너 | 클로저 참조 유지 | `removeEventListener` 또는 `AbortController` |
| 타이머 | 콜백과 참조 객체 유지 | `clearInterval` / `clearTimeout` |
| 대형 클로저 | 불필요한 객체 참조 | 필요한 값만 추출해 참조 |
| 캐시 Map | 키 객체 참조 유지 | `WeakMap` / `WeakRef` 활용 |

JavaScript 메모리 관리는 처음에는 복잡하게 느껴질 수 있지만, 위의 패턴만 숙지해도 대부분의 누수 문제를 예방할 수 있습니다. Chrome DevTools의 Memory 탭을 주기적으로 모니터링하는 습관을 들이면 성능 문제를 조기에 발견하고 더 안정적인 애플리케이션을 만들 수 있습니다.
