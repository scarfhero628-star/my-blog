---
title: 실무에서 자주 쓰는 JavaScript 비동기 패턴 5가지
date: 2026-09-04
tags: [JavaScript, 비동기, async/await, Promise]
description: 콜백 지옥을 탈출하고 가독성 높은 코드를 짜기 위해 알아야 할 비동기 패턴을 정리했습니다.
---

## 들어가며

JavaScript는 싱글 스레드 언어이지만, 비동기 처리 덕분에 네트워크 요청이나 파일 I/O 같은 시간이 걸리는 작업도 효율적으로 다룰 수 있습니다. 그런데 비동기 코드는 잘못 쓰면 디버깅이 어렵고, 읽기도 힘들어집니다. 이 글에서는 실무에서 자주 마주치는 비동기 패턴 다섯 가지를 예제와 함께 살펴봅니다.

---

## 1. async/await 기본 패턴

`Promise`를 직접 `.then()`으로 체이닝하는 것보다 `async/await`를 쓰면 동기 코드처럼 읽힙니다.

```js
async function fetchUser(id) {
  const response = await fetch(`/api/users/${id}`);
  if (!response.ok) throw new Error('유저 조회 실패');
  return response.json();
}
```

### 에러 처리

`try/catch`와 함께 쓰면 에러 흐름이 명확해집니다.

```js
async function loadProfile(id) {
  try {
    const user = await fetchUser(id);
    return user;
  } catch (err) {
    console.error('프로필 로드 오류:', err.message);
    return null;
  }
}
```

---

## 2. 병렬 실행: Promise.all

여러 비동기 작업이 서로 의존하지 않는다면, 순차 실행 대신 `Promise.all`로 동시에 처리하세요. 속도가 크게 향상됩니다.

```js
// 나쁜 예: 순차 실행 (총 500ms 소요)
const posts = await fetchPosts();
const comments = await fetchComments();

// 좋은 예: 병렬 실행 (총 약 250ms 소요)
const [posts, comments] = await Promise.all([
  fetchPosts(),
  fetchComments(),
]);
```

`Promise.all`은 하나라도 실패하면 전체가 reject됩니다. 실패를 개별로 처리하고 싶다면 `Promise.allSettled`를 씁니다.

```js
const results = await Promise.allSettled([fetchPosts(), fetchComments()]);

results.forEach((result) => {
  if (result.status === 'fulfilled') {
    console.log('성공:', result.value);
  } else {
    console.warn('실패:', result.reason);
  }
});
```

---

## 3. 경쟁: Promise.race

여러 소스 중 가장 빨리 응답한 것을 쓰거나, 타임아웃을 구현할 때 유용합니다.

```js
function withTimeout(promise, ms) {
  const timeout = new Promise((_, reject) =>
    setTimeout(() => reject(new Error('요청 시간 초과')), ms)
  );
  return Promise.race([promise, timeout]);
}

const data = await withTimeout(fetch('/api/slow-endpoint'), 3000);
```

---

## 4. 순차 처리가 필요할 때: for...of + await

배열의 각 항목을 순서대로 처리해야 할 때 `Array.map` 안에서 `await`를 쓰면 의도대로 동작하지 않습니다. `for...of`를 사용하세요.

```js
// 의도와 다른 코드: 모두 동시에 실행됨
const results = await Promise.all(ids.map(id => processItem(id)));

// 순차 처리가 꼭 필요한 경우
for (const id of ids) {
  await processItem(id); // 이전 작업이 끝난 후 다음으로 넘어감
}
```

순차가 꼭 필요한 상황의 예시: 이전 결과값을 다음 요청에 사용해야 하거나, API rate limit이 있어 동시 요청을 제한해야 할 때.

---

## 5. 재시도 패턴 (Retry)

네트워크 오류처럼 일시적인 실패를 자동으로 재시도하는 패턴입니다.

```js
async function fetchWithRetry(url, retries = 3, delay = 500) {
  for (let attempt = 1; attempt <= retries; attempt++) {
    try {
      const res = await fetch(url);
      if (!res.ok) throw new Error(`HTTP ${res.status}`);
      return await res.json();
    } catch (err) {
      if (attempt === retries) throw err;
      console.warn(`재시도 ${attempt}/${retries}...`);
      await new Promise((r) => setTimeout(r, delay * attempt));
    }
  }
}
```

재시도 간격을 점점 늘리는 **지수 백오프(Exponential Backoff)** 방식을 적용하면 서버 부하를 줄일 수 있습니다.

---

## 정리

| 패턴 | 사용 시점 |
|---|---|
| `async/await` | 기본 비동기 흐름 |
| `Promise.all` | 독립적인 작업을 병렬 실행 |
| `Promise.allSettled` | 실패해도 나머지 결과가 필요할 때 |
| `Promise.race` | 타임아웃, 가장 빠른 응답 선택 |
| `for...of + await` | 순서가 중요한 순차 처리 |
| 재시도 패턴 | 일시적 네트워크 오류 대응 |

비동기 코드는 패턴만 잘 골라도 가독성과 안정성이 크게 올라갑니다. 처음부터 완벽하게 쓰려 하기보다, 일단 `async/await`로 시작해서 필요에 따라 `Promise.all`이나 재시도 패턴을 추가하는 방식을 권장합니다.
