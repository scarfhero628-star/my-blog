---
title: JavaScript 함수형 프로그래밍 입문 — 부작용을 줄이고 예측 가능한 코드 작성하기
date: 2026-10-03
tags: [함수형 프로그래밍, JavaScript, 웹 개발, 코드 품질, 순수 함수]
description: 순수 함수·불변성·고차 함수 등 함수형 프로그래밍의 핵심 개념을 실전 예제와 함께 익히고 더 예측 가능한 JavaScript 코드를 작성하는 방법을 소개합니다.
---

## 왜 함수형 프로그래밍인가?

객체지향 프로그래밍이 익숙한 개발자라면 "굳이 함수형을 배워야 하나?"라고 생각할 수 있습니다. 하지만 함수형 프로그래밍(Functional Programming, FP)이 주목받는 이유는 명확합니다. **코드의 예측 가능성**과 **버그 발생 가능성을 줄이는 구조**를 제공하기 때문입니다.

JavaScript는 멀티 패러다임 언어입니다. 객체지향과 함수형을 동시에 지원하므로 FP의 핵심 원칙만 이해해도 기존 코드 품질을 크게 높일 수 있습니다.

---

## 1. 순수 함수 (Pure Function)

순수 함수는 함수형 프로그래밍의 가장 기본 개념입니다. 두 가지 조건을 만족해야 합니다.

1. **동일한 입력에는 항상 동일한 출력**
2. **부작용(Side Effect) 없음** — 함수 외부 상태를 변경하지 않음

```js
// 순수하지 않은 함수 — 외부 상태에 의존
let discount = 10;
function getPrice(price) {
  return price - discount; // discount가 바뀌면 결과도 바뀜
}

// 순수 함수
function getPrice(price, discount) {
  return price - discount; // 항상 같은 입력 → 같은 출력
}
```

순수 함수는 테스트하기 쉽고, 동작을 예측하기 쉽습니다. 함수 하나만 떼어내도 독립적으로 검증할 수 있기 때문입니다.

---

## 2. 불변성 (Immutability)

함수형 프로그래밍에서는 데이터를 **변경(mutate)하지 않고 새로운 데이터를 반환**합니다. 기존 객체나 배열을 직접 수정하면 어디서 상태가 바뀌었는지 추적하기 어렵습니다.

```js
// 나쁜 예 — 원본 배열 변형
const todos = ['밥 먹기', '운동하기'];
todos.push('코딩하기'); // 원본이 바뀜

// 좋은 예 — 새 배열 반환
const todos = ['밥 먹기', '운동하기'];
const newTodos = [...todos, '코딩하기']; // 원본 유지

// 객체도 마찬가지
const user = { name: '홍길동', age: 30 };
const updatedUser = { ...user, age: 31 }; // 새 객체 생성
```

React를 사용해봤다면 이미 익숙한 패턴입니다. React의 상태 업데이트 원칙 자체가 불변성에 기반합니다.

---

## 3. 고차 함수 (Higher-Order Function)

고차 함수는 **함수를 인자로 받거나 함수를 반환하는 함수**입니다. JavaScript 배열의 `map`, `filter`, `reduce`가 대표적인 고차 함수입니다.

```js
const products = [
  { name: '노트북', price: 1200000, inStock: true },
  { name: '마우스', price: 35000, inStock: false },
  { name: '키보드', price: 120000, inStock: true },
];

// 재고 있는 상품만 필터링 후 이름만 추출
const availableNames = products
  .filter(p => p.inStock)
  .map(p => p.name);
// ['노트북', '키보드']

// 재고 있는 상품의 총 가격 합산
const totalPrice = products
  .filter(p => p.inStock)
  .reduce((sum, p) => sum + p.price, 0);
// 1320000
```

`for` 루프 대신 `map/filter/reduce`를 사용하면 **코드 의도가 명확**해집니다. 어떤 데이터를 어떻게 변환하는지 한눈에 파악할 수 있습니다.

---

## 4. 함수 합성 (Function Composition)

여러 작은 함수를 조합해 복잡한 로직을 구성하는 기법입니다. 유닉스 파이프(`|`)와 비슷한 발상입니다.

```js
const trim = str => str.trim();
const toLower = str => str.toLowerCase();
const removeSpaces = str => str.replace(/\s+/g, '-');

// 직접 중첩 호출 — 읽기 불편
const slugify = str => removeSpaces(toLower(trim(str)));

// compose 유틸 함수 활용
const compose = (...fns) => x => fns.reduceRight((v, f) => f(v), x);
const slugify = compose(removeSpaces, toLower, trim);

console.log(slugify('  Hello World  ')); // 'hello-world'
```

각 함수가 단일 책임을 지고, 조합으로 복잡한 변환을 표현할 수 있습니다. 함수 단위의 테스트도 훨씬 쉬워집니다.

---

## 5. 커링 (Currying)

커링은 여러 인자를 받는 함수를 **인자 하나씩 받는 함수의 연쇄**로 변환하는 기법입니다. 부분 적용(Partial Application)을 가능하게 해 코드 재사용성을 높입니다.

```js
// 일반 함수
function multiply(a, b) {
  return a * b;
}

// 커링된 함수
const multiply = a => b => a * b;

const double = multiply(2);  // 2를 고정
const triple = multiply(3);  // 3을 고정

console.log(double(5));  // 10
console.log(triple(5));  // 15

// 실전 예 — 로그 함수
const log = level => message => `[${level}] ${message}`;
const warn = log('WARN');
const error = log('ERROR');

warn('디스크 공간 부족');  // '[WARN] 디스크 공간 부족'
error('서버 연결 실패');   // '[ERROR] 서버 연결 실패'
```

---

## 6. 실전 적용 — 함수형으로 API 데이터 처리하기

아래는 API 응답 데이터를 가공하는 흔한 상황입니다. 명령형 vs 함수형으로 비교해봅니다.

```js
// 명령형 방식
function processUsers(users) {
  const result = [];
  for (let i = 0; i < users.length; i++) {
    if (users[i].isActive) {
      result.push({
        id: users[i].id,
        name: users[i].name.trim(),
        email: users[i].email.toLowerCase(),
      });
    }
  }
  return result;
}

// 함수형 방식
const isActive = user => user.isActive;
const normalize = user => ({
  id: user.id,
  name: user.name.trim(),
  email: user.email.toLowerCase(),
});

const processUsers = users => users.filter(isActive).map(normalize);
```

함수형 방식은 더 짧고, 각 단계의 의도가 명확하며, `isActive`와 `normalize` 함수를 다른 곳에서도 재사용할 수 있습니다.

---

## 핵심 정리

| 개념 | 핵심 원칙 |
|------|-----------|
| 순수 함수 | 외부 상태 의존·변경 금지, 동일 입력 → 동일 출력 |
| 불변성 | 데이터를 직접 수정하지 않고 새 값을 반환 |
| 고차 함수 | 함수를 인자로 받거나 반환하는 함수 (`map`, `filter`, `reduce`) |
| 함수 합성 | 작은 함수들을 조합해 복잡한 로직 구성 |
| 커링 | 인자를 나눠 받아 부분 적용 및 재사용성 향상 |

함수형 프로그래밍을 처음 접하면 낯설게 느껴질 수 있습니다. 하지만 모든 코드를 한 번에 함수형으로 바꿀 필요는 없습니다. **순수 함수부터 시작해** 불변성을 지키는 습관을 들이고, 자연스럽게 `map/filter/reduce`를 활용하다 보면 코드가 훨씬 깔끔해지는 것을 느낄 수 있습니다.
