---
title: 클린 코드 작성을 위한 5가지 핵심 원칙 — 읽기 좋은 코드가 좋은 코드다
date: 2026-09-21
tags: [클린 코드, 코드 품질, 리팩토링, 개발자 습관]
description: 동료가 이해하기 쉽고 유지보수하기 좋은 코드를 작성하기 위한 5가지 클린 코드 원칙과 실전 예제를 소개합니다.
---

## 왜 클린 코드가 중요한가

코드는 한 번 쓰고 끝나지 않습니다. 우리가 실제로 코드를 작성하는 시간보다 **기존 코드를 읽는 시간이 훨씬 길다**는 사실은 여러 연구와 현장 개발자들의 경험이 공통적으로 증명하고 있습니다. 로버트 마틴(Robert C. Martin)은 그의 저서 *Clean Code*에서 "코드를 읽는 시간 대 쓰는 시간의 비율은 10:1 이상"이라고 말했습니다.

결국 클린 코드란 단순히 "예쁜 코드"가 아닙니다. 미래의 나 자신과 팀원이 빠르게 이해하고, 안전하게 수정할 수 있도록 배려한 코드입니다. 이 글에서는 현장에서 바로 적용할 수 있는 5가지 핵심 원칙을 소개합니다.

---

## 원칙 1. 의도를 드러내는 이름을 짓자

가장 강력하면서도 즉시 실천할 수 있는 원칙입니다. 변수명, 함수명, 클래스명은 **코드가 무엇을 하는지**가 아닌 **왜 존재하는지**를 담아야 합니다.

### 나쁜 예

```javascript
// d는 무엇인가? 날짜인가 데이터인가?
const d = new Date();

// 이 함수는 무엇을 반환하나?
function check(u) {
  return u.age >= 18;
}
```

### 좋은 예

```javascript
const currentDate = new Date();

function isAdult(user) {
  return user.age >= 18;
}
```

이름 짓기가 어렵다면, 그 코드의 역할이 아직 명확하지 않다는 신호일 수 있습니다. 좋은 이름을 찾는 과정 자체가 설계를 정리하는 행위입니다.

---

## 원칙 2. 함수는 한 가지 일만 해야 한다

**단일 책임 원칙(SRP, Single Responsibility Principle)**은 함수 레벨에서도 동일하게 적용됩니다. 함수가 여러 일을 동시에 하면 테스트하기 어렵고, 이름 짓기도 어려워집니다.

### 나쁜 예

```javascript
async function processUserOrder(userId, orderId) {
  // 사용자 검증
  const user = await getUser(userId);
  if (!user || !user.isActive) throw new Error('Invalid user');

  // 재고 확인
  const order = await getOrder(orderId);
  if (order.stock < 1) throw new Error('Out of stock');

  // 결제 처리
  await chargePayment(user.paymentInfo, order.price);

  // 이메일 발송
  await sendConfirmationEmail(user.email, order);
}
```

### 좋은 예

```javascript
async function processUserOrder(userId, orderId) {
  const user = await validateUser(userId);
  const order = await validateOrderStock(orderId);
  await chargePayment(user.paymentInfo, order.price);
  await sendConfirmationEmail(user.email, order);
}
```

각 검증과 처리를 별도 함수로 분리하면 각 단계를 독립적으로 테스트하고 재사용할 수 있습니다.

---

## 원칙 3. 주석보다 코드 자체를 읽기 쉽게 만들어라

주석은 **왜(Why)** 이 코드가 필요한지 설명할 때만 사용해야 합니다. **무엇을(What)** 하는지 설명하는 주석은 코드 자체가 불명확하다는 증거입니다.

### 주석이 필요 없는 코드

```javascript
// 나쁜 예: 주석이 코드를 설명하고 있다
// 사용자 목록에서 활성 상태이고 나이가 18세 이상인 사용자만 필터링
const result = users.filter(u => u.status === 1 && u.age >= 18);

// 좋은 예: 코드 자체가 의미를 전달한다
const ACTIVE_STATUS = 1;
const eligibleUsers = users.filter(user =>
  user.status === ACTIVE_STATUS && isAdult(user)
);
```

### 주석이 꼭 필요한 경우

```javascript
// 특정 브라우저의 버그로 인해 setTimeout 0이 필요합니다 (IE11 이슈 #4521)
setTimeout(() => renderComponent(), 0);
```

외부 제약, 알려진 버그 우회, 비직관적인 비즈니스 로직 등 **이유(Why)**가 있는 주석은 미래의 개발자에게 소중한 정보가 됩니다.

---

## 원칙 4. 중복을 제거하라 (DRY 원칙)

**DRY(Don't Repeat Yourself)** 원칙은 클린 코드의 기본 중 기본입니다. 같은 로직이 두 곳 이상에 있다면, 한 곳에서 버그를 수정해도 다른 곳은 그대로 남게 됩니다.

### 중복 코드의 위험

```javascript
// 나쁜 예: 같은 날짜 포맷 로직이 곳곳에 반복됨
function displayCreatedAt(post) {
  const date = new Date(post.createdAt);
  return `${date.getFullYear()}-${String(date.getMonth() + 1).padStart(2, '0')}-${String(date.getDate()).padStart(2, '0')}`;
}

function displayUpdatedAt(post) {
  const date = new Date(post.updatedAt);
  return `${date.getFullYear()}-${String(date.getMonth() + 1).padStart(2, '0')}-${String(date.getDate()).padStart(2, '0')}`;
}
```

### 중복 제거 후

```javascript
// 좋은 예: 공통 로직을 한 곳으로
function formatDate(isoString) {
  const date = new Date(isoString);
  return `${date.getFullYear()}-${String(date.getMonth() + 1).padStart(2, '0')}-${String(date.getDate()).padStart(2, '0')}`;
}

const displayCreatedAt = (post) => formatDate(post.createdAt);
const displayUpdatedAt = (post) => formatDate(post.updatedAt);
```

단, 모든 중복 제거가 좋은 것은 아닙니다. 겉보기에 비슷한 코드라도 **우연의 일치**라면 억지로 추상화하지 않는 것이 좋습니다. "세 번 반복될 때" 추상화를 고려하는 **삼진 규칙(Rule of Three)**도 좋은 기준입니다.

---

## 원칙 5. 작은 단위로 나누고, 들여쓰기를 최소화하라

함수가 깊은 들여쓰기를 가지면 읽기 어렵고, 테스트하기도 어렵습니다. **조기 반환(Early Return)** 패턴과 함수 분리로 들여쓰기 깊이를 줄일 수 있습니다.

### 들여쓰기가 깊은 코드

```javascript
function getDiscount(user) {
  if (user) {
    if (user.isLoggedIn) {
      if (user.membership === 'premium') {
        if (user.purchaseCount > 10) {
          return 0.3;
        } else {
          return 0.2;
        }
      } else {
        return 0.1;
      }
    }
  }
  return 0;
}
```

### 조기 반환으로 개선한 코드

```javascript
function getDiscount(user) {
  if (!user || !user.isLoggedIn) return 0;
  if (user.membership !== 'premium') return 0.1;
  if (user.purchaseCount > 10) return 0.3;
  return 0.2;
}
```

같은 로직이지만 훨씬 읽기 쉽습니다. 조건이 위쪽에서 걸러지기 때문에 아래로 내려갈수록 좁은 케이스를 다룬다는 사실을 직관적으로 파악할 수 있습니다.

---

## 클린 코드를 실천하는 방법

원칙을 아는 것과 꾸준히 실천하는 것은 다른 이야기입니다. 몇 가지 실용적인 팁을 드립니다.

- **보이스카우트 규칙**: 캠프를 처음 왔을 때보다 깨끗하게 놔두고 가라. 코드를 열 때마다 조금씩 개선합니다.
- **코드 리뷰를 통한 피드백**: 동료의 시선으로 내 코드를 점검하는 가장 효과적인 방법입니다.
- **리팩토링 시간 확보**: 새 기능 개발과 별도로 리팩토링을 위한 시간을 의도적으로 확보합니다.
- **정기적인 독서**: *Clean Code*(Robert C. Martin), *The Pragmatic Programmer*, *Refactoring*(Martin Fowler) 같은 고전을 읽고 팀과 함께 토론합니다.

---

## 마치며

클린 코드는 한 번에 완성되지 않습니다. 끊임없이 다듬고, 리뷰하고, 개선하는 **과정**입니다. 오늘 당장 모든 원칙을 적용하려 하기보다, 하나씩 습관으로 만들어 나가는 것이 중요합니다. 읽기 좋은 코드를 쓰는 것은 결국 팀과 미래의 자신에 대한 배려입니다.
