---
title: JavaScript 에러 처리 완전 가이드 — 견고한 애플리케이션을 위한 오류 처리 패턴
date: 2026-11-13
tags: [JavaScript, 에러 처리, 비동기, Node.js, 웹 개발]
description: 동기·비동기 환경에서 JavaScript 에러를 체계적으로 처리하는 핵심 패턴과 실전 전략을 소개합니다.
---

에러 처리는 코드를 처음 짤 때 가장 미루고 싶은 작업입니다. 하지만 운영 중인 서비스에서 처리되지 않은 에러 하나가 사용자 경험 전체를 망가뜨릴 수 있습니다. 이 글에서는 JavaScript에서 에러를 체계적으로 처리하는 패턴을 동기·비동기 환경으로 나눠 살펴봅니다.

## 1. 동기 에러 처리 — try / catch / finally

가장 기본적인 에러 처리는 `try-catch` 블록입니다. `finally`는 에러 발생 여부와 관계없이 반드시 실행됩니다.

```js
function parseJSON(str) {
  try {
    return JSON.parse(str);
  } catch (err) {
    console.error('JSON 파싱 실패:', err.message);
    return null;
  } finally {
    console.log('파싱 시도 완료');
  }
}
```

### 주의: catch는 예상 가능한 에러에만 사용

모든 `catch` 블록에서 에러를 조용히 삼키는 패턴은 위험합니다. 예상하지 못한 에러까지 묻어버리면 디버깅이 불가능해집니다.

```js
// 나쁜 예
try {
  riskyOperation();
} catch (e) {
  // 아무것도 하지 않음
}

// 좋은 예: 최소한 로깅은 하고, 복구 불가능한 에러는 다시 던지기
try {
  riskyOperation();
} catch (e) {
  if (e instanceof NetworkError) {
    return fallbackData;
  }
  throw e; // 처리할 수 없는 에러는 위로 전파
}
```

## 2. 비동기 에러 처리

### Promise 체인에서의 에러 처리

`.catch()`를 체인 끝에 한 번만 붙이면 체인 어디서든 발생한 에러를 잡을 수 있습니다.

```js
fetchUser(userId)
  .then(user => fetchOrders(user.id))
  .then(orders => renderOrders(orders))
  .catch(err => {
    console.error('주문 불러오기 실패:', err);
    showErrorToast('데이터를 불러올 수 없습니다.');
  });
```

### async/await와 try-catch

`async/await`를 사용하면 비동기 코드도 동기처럼 에러를 처리할 수 있습니다.

```js
async function loadUserDashboard(userId) {
  try {
    const user = await fetchUser(userId);
    const orders = await fetchOrders(user.id);
    return { user, orders };
  } catch (err) {
    if (err.status === 404) {
      return { user: null, orders: [] };
    }
    throw err;
  }
}
```

### 여러 Promise를 동시에 실행할 때

`Promise.all`은 하나라도 실패하면 전체가 실패합니다. 개별 에러를 처리하려면 `Promise.allSettled`를 사용하세요.

```js
const results = await Promise.allSettled([
  fetchUser(userId),
  fetchNotifications(userId),
  fetchRecommendations(userId),
]);

results.forEach(result => {
  if (result.status === 'fulfilled') {
    console.log('성공:', result.value);
  } else {
    console.warn('실패:', result.reason);
  }
});
```

## 3. 커스텀 에러 클래스

내장 `Error`를 확장하면 에러 유형을 구분하고, 추가 컨텍스트를 담을 수 있습니다.

```js
class AppError extends Error {
  constructor(message, code, statusCode = 500) {
    super(message);
    this.name = 'AppError';
    this.code = code;
    this.statusCode = statusCode;
  }
}

class NotFoundError extends AppError {
  constructor(resource) {
    super(`${resource}을(를) 찾을 수 없습니다.`, 'NOT_FOUND', 404);
    this.name = 'NotFoundError';
  }
}

class ValidationError extends AppError {
  constructor(field, message) {
    super(message, 'VALIDATION_ERROR', 400);
    this.name = 'ValidationError';
    this.field = field;
  }
}
```

이렇게 정의하면 `instanceof`로 에러 종류를 명확하게 분기할 수 있습니다.

```js
try {
  const user = await getUser(id);
} catch (err) {
  if (err instanceof NotFoundError) {
    return res.status(404).json({ message: err.message });
  }
  if (err instanceof ValidationError) {
    return res.status(400).json({ field: err.field, message: err.message });
  }
  // 예상 못한 에러는 500으로 처리
  return res.status(500).json({ message: '서버 오류가 발생했습니다.' });
}
```

## 4. 전역 에러 처리

### 브라우저 환경

```js
// 동기 에러
window.addEventListener('error', (event) => {
  console.error('전역 에러:', event.error);
  sendErrorToMonitoring(event.error);
});

// 처리되지 않은 Promise rejection
window.addEventListener('unhandledrejection', (event) => {
  console.error('미처리 Promise 에러:', event.reason);
  event.preventDefault(); // 콘솔 출력 방지
});
```

### Node.js 환경

```js
process.on('uncaughtException', (err) => {
  console.error('처리되지 않은 예외:', err);
  // 로그 기록 후 프로세스 종료 (상태가 불안정할 수 있음)
  process.exit(1);
});

process.on('unhandledRejection', (reason) => {
  console.error('미처리 Promise rejection:', reason);
});
```

## 5. 실전 베스트 프랙티스

다음 원칙을 지키면 에러 처리 코드의 품질을 크게 높일 수 있습니다.

- **에러를 조용히 삼키지 마세요.** 최소한 로그를 남겨야 디버깅이 가능합니다.
- **복구 불가능한 에러는 위로 전파하세요.** 각 계층에서 처리할 수 있는 에러만 처리하고, 나머지는 던지세요.
- **커스텀 에러 클래스로 의미를 명확히 하세요.** `Error: failed`보다 `NotFoundError: user를 찾을 수 없습니다.`가 훨씬 유용합니다.
- **에러 모니터링 도구를 연동하세요.** Sentry, Datadog 등을 사용하면 운영 환경의 에러를 실시간으로 추적할 수 있습니다.
- **에러 메시지에 컨텍스트를 담으세요.** 어떤 작업을 하다 실패했는지, 어떤 입력값이 문제였는지를 메시지에 포함하면 디버깅 시간이 크게 줄어듭니다.

## 마치며

에러 처리는 단순히 앱이 죽지 않도록 하는 것이 아닙니다. 사용자에게 적절한 피드백을 제공하고, 개발자가 문제를 빠르게 파악할 수 있도록 충분한 정보를 남기는 것이 핵심입니다. 오늘 소개한 패턴을 프로젝트에 적용하면 더 견고하고 디버깅하기 쉬운 애플리케이션을 만들 수 있습니다.
