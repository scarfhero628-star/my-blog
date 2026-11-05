---
title: JavaScript 제너레이터와 이터레이터 완전 가이드 — 지연 평가로 성능을 높이는 비동기 패턴
date: 2026-11-05
tags: [JavaScript, 제너레이터, 이터레이터, 비동기, 웹 개발]
description: JavaScript 제너레이터와 이터레이터의 동작 원리를 이해하고, 지연 평가·무한 시퀀스·비동기 흐름 제어에 활용하는 실전 패턴을 소개합니다.
---

JavaScript를 어느 정도 다뤄본 개발자라면 `for...of`, 스프레드 연산자, 구조 분해 할당을 자연스럽게 쓰고 있을 겁니다. 그런데 이 모든 문법이 하나의 공통된 프로토콜 위에서 동작한다는 사실을 알고 있나요? 바로 **이터레이터 프로토콜(Iterator Protocol)** 입니다. 여기에 제너레이터(Generator)를 더하면 지연 평가, 무한 시퀀스, 비동기 흐름 제어까지 우아하게 처리할 수 있습니다. 오늘은 이 두 가지 개념을 뿌리부터 이해하고 실무에 바로 적용하는 방법을 소개합니다.

## 이터레이터 프로토콜이란?

JavaScript의 이터레이터는 `next()` 메서드를 가진 객체입니다. `next()`를 호출할 때마다 `{ value, done }` 형태의 객체를 반환합니다.

```javascript
function makeCounter(start = 0) {
  let current = start;
  return {
    next() {
      return { value: current++, done: false };
    },
  };
}

const counter = makeCounter(1);
console.log(counter.next()); // { value: 1, done: false }
console.log(counter.next()); // { value: 2, done: false }
```

`done: true`를 반환하면 순회가 끝납니다. 이터러블(Iterable)은 `Symbol.iterator` 메서드를 구현한 객체로, `for...of`와 스프레드 연산자가 이 메서드를 호출해 이터레이터를 얻습니다.

```javascript
const range = {
  from: 1,
  to: 5,
  [Symbol.iterator]() {
    let current = this.from;
    const last = this.to;
    return {
      next() {
        return current <= last
          ? { value: current++, done: false }
          : { value: undefined, done: true };
      },
    };
  },
};

for (const num of range) {
  console.log(num); // 1 2 3 4 5
}

console.log([...range]); // [1, 2, 3, 4, 5]
```

## 제너레이터 함수: 이터레이터를 쉽게 만드는 방법

이터레이터를 직접 구현하면 보일러플레이트가 많습니다. 제너레이터 함수(`function*`)를 사용하면 훨씬 간결하게 작성할 수 있습니다.

```javascript
function* range(from, to) {
  for (let i = from; i <= to; i++) {
    yield i; // 값을 하나씩 내보낸다
  }
}

for (const num of range(1, 5)) {
  console.log(num); // 1 2 3 4 5
}
```

`yield` 키워드가 핵심입니다. 함수 실행을 일시 중지하고 값을 반환한 뒤, 다음 `next()` 호출 시 중단된 지점부터 재개합니다. 일반 함수와 달리 **실행 컨텍스트가 유지**된다는 점이 중요합니다.

### 제너레이터의 양방향 통신

`next(value)`에 인자를 전달하면 제너레이터 내부에서 `yield` 표현식의 반환값으로 받을 수 있습니다.

```javascript
function* calculator() {
  let result = 0;
  while (true) {
    const input = yield result; // yield는 현재 result를 반환하고, 다음 next(value)의 value를 받는다
    if (input === null) break;
    result += input;
  }
  return result;
}

const calc = calculator();
calc.next();       // 초기화 { value: 0, done: false }
calc.next(10);     // { value: 10, done: false }
calc.next(5);      // { value: 15, done: false }
calc.next(null);   // { value: 15, done: true }
```

## 실전 패턴 1: 무한 시퀀스와 지연 평가

가장 강력한 활용 사례 중 하나는 **무한 시퀀스**입니다. 메모리에 모든 값을 올리지 않고 필요할 때마다 하나씩 계산합니다.

```javascript
function* fibonacci() {
  let [a, b] = [0, 1];
  while (true) {
    yield a;
    [a, b] = [b, a + b];
  }
}

function take(n, iterable) {
  const result = [];
  for (const val of iterable) {
    result.push(val);
    if (result.length >= n) break;
  }
  return result;
}

console.log(take(8, fibonacci())); // [0, 1, 1, 2, 3, 5, 8, 13]
```

배열을 미리 생성하지 않고 필요한 만큼만 소비합니다. 대용량 데이터 처리나 페이지네이션 구현에 매우 유용합니다.

## 실전 패턴 2: 지연 파이프라인

`map`, `filter`를 배열에 연달아 쓰면 각 단계마다 중간 배열이 생성됩니다. 제너레이터를 활용하면 중간 배열 없이 값을 하나씩 흘려보내는 파이프라인을 만들 수 있습니다.

```javascript
function* map(iterable, fn) {
  for (const val of iterable) {
    yield fn(val);
  }
}

function* filter(iterable, predicate) {
  for (const val of iterable) {
    if (predicate(val)) yield val;
  }
}

// 1~1000 중 짝수의 제곱을 처음 5개만 가져오기
const result = take(
  5,
  map(
    filter(range(1, 1000), (n) => n % 2 === 0),
    (n) => n * n
  )
);

console.log(result); // [4, 16, 36, 64, 100]
```

중간 배열이 전혀 없이 필요한 5개만 계산하고 멈춥니다. 1000개 원소를 다 처리하지 않아도 됩니다.

## 실전 패턴 3: 비동기 흐름 제어

제너레이터는 `async/await`의 초기 대안으로 사용됐습니다. 오늘날에도 복잡한 비동기 오케스트레이션이나 Redux-Saga 같은 라이브러리에서 핵심 역할을 합니다.

```javascript
function run(generatorFn) {
  const gen = generatorFn();

  function step(value) {
    const { value: promise, done } = gen.next(value);
    if (!done) {
      Promise.resolve(promise).then(step);
    }
  }

  step();
}

function* fetchUser(id) {
  const user = yield fetch(`/api/users/${id}`).then((r) => r.json());
  const posts = yield fetch(`/api/posts?userId=${user.id}`).then((r) =>
    r.json()
  );
  console.log(user, posts);
}

run(() => fetchUser(1));
```

이 패턴이 `async/await`의 근간입니다. `async function`은 사실 제너레이터 + Promise를 자동으로 처리해주는 문법 설탕입니다.

## 실전 패턴 4: 상태 머신

제너레이터는 실행 상태를 함수 내부에 자연스럽게 보존하기 때문에 간단한 상태 머신 구현에도 적합합니다.

```javascript
function* trafficLight() {
  while (true) {
    yield '🔴 빨간불 — 정지';
    yield '🟡 노란불 — 준비';
    yield '🟢 초록불 — 출발';
  }
}

const light = trafficLight();
console.log(light.next().value); // 🔴 빨간불 — 정지
console.log(light.next().value); // 🟡 노란불 — 준비
console.log(light.next().value); // 🟢 초록불 — 출발
console.log(light.next().value); // 🔴 빨간불 — 정지 (다시 반복)
```

외부 변수 없이 상태 전이를 순서대로 표현할 수 있습니다.

## 주의할 점

제너레이터를 쓸 때 알아두면 좋은 주의 사항입니다.

- **일회용**: 제너레이터 객체는 한 번 소진되면 재사용할 수 없습니다. 필요하면 새로 호출하세요.
- **조기 종료**: `return()` 메서드를 호출하면 `done: true`가 되며 `try...finally`의 `finally` 블록이 실행됩니다.
- **에러 전달**: `throw()` 메서드로 제너레이터 내부에 에러를 주입할 수 있으며 `try...catch`로 처리할 수 있습니다.
- **성능**: 단순 배열 순회에는 제너레이터보다 `for` 루프가 빠릅니다. 지연 평가가 의미있는 상황, 즉 큰 데이터셋이나 무한 시퀀스일 때 사용하세요.

## 정리

| 개념 | 핵심 용도 |
|------|-----------|
| 이터레이터 | 커스텀 순회 로직 구현 |
| 이터러블 | `for...of`, 스프레드, 구조 분해 지원 |
| 제너레이터 | 이터레이터 간편 작성, 지연 평가 |
| 무한 시퀀스 | 메모리 효율적 대용량 처리 |
| 비동기 제너레이터 | 스트리밍 데이터, 페이지네이션 |

이터레이터와 제너레이터를 이해하면 JavaScript의 순회 프로토콜 전반을 꿰뚫게 됩니다. 처음에는 낯설게 느껴질 수 있지만, 한 번 익숙해지면 대용량 데이터 처리, 비동기 오케스트레이션, 상태 머신 등 다양한 상황에서 코드를 훨씬 명확하고 효율적으로 작성할 수 있게 됩니다.
