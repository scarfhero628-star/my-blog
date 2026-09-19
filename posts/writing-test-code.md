---
title: 개발자를 위한 테스트 코드 작성법 — 믿을 수 있는 코드를 만드는 5가지 원칙
date: 2026-09-19
tags: [테스팅, 단위 테스트, TDD, 코드 품질]
description: 테스트 코드를 처음 시작하거나 더 잘 작성하고 싶은 개발자를 위한 실전 5가지 원칙을 소개합니다.
---

## 왜 테스트 코드를 작성해야 하는가

코드를 짜고 브라우저나 터미널에서 직접 확인하는 방식은 처음엔 빠르게 느껴집니다. 하지만 프로젝트가 커질수록 수동 테스트는 한계를 드러냅니다. 새 기능을 추가했을 때 기존 기능이 망가졌는지 매번 전부 확인할 수 없고, 배포 전날 밤 긴장감은 점점 커집니다.

잘 작성된 테스트 코드는 이 문제를 해결합니다. 변경할 때마다 자동으로 실행되어 회귀(regression)를 잡아주고, 코드가 어떻게 동작해야 하는지를 문서처럼 설명해줍니다. **테스트는 코드에 대한 자신감입니다.**

## 좋은 테스트의 5가지 원칙

### 1. 하나의 테스트는 하나의 동작만 검증한다

테스트가 실패했을 때 무엇이 문제인지 바로 알 수 있어야 합니다. 하나의 테스트에 여러 동작을 넣으면 실패 원인을 파악하는 데 시간이 걸립니다.

```javascript
// 나쁜 예: 여러 동작을 한 테스트에 넣기
test('사용자 관련 기능', () => {
  const user = createUser('Alice', 'alice@example.com');
  expect(user.name).toBe('Alice');
  expect(user.email).toBe('alice@example.com');
  expect(user.isActive).toBe(true);
  expect(sendWelcomeEmail).toHaveBeenCalled();
});

// 좋은 예: 동작별로 분리
test('사용자 생성 시 이름이 설정된다', () => {
  const user = createUser('Alice', 'alice@example.com');
  expect(user.name).toBe('Alice');
});

test('사용자 생성 시 환영 이메일이 발송된다', () => {
  createUser('Alice', 'alice@example.com');
  expect(sendWelcomeEmail).toHaveBeenCalledWith('alice@example.com');
});
```

### 2. AAA 패턴으로 테스트를 구조화한다

**Arrange → Act → Assert** 패턴을 따르면 테스트 코드가 읽기 쉬워집니다.

- **Arrange**: 테스트에 필요한 데이터와 환경을 준비한다
- **Act**: 테스트 대상 코드를 실행한다
- **Assert**: 결과가 예상과 일치하는지 확인한다

```javascript
test('장바구니에 상품을 추가하면 총액이 증가한다', () => {
  // Arrange
  const cart = new ShoppingCart();
  const item = { name: '노트북', price: 1200000 };

  // Act
  cart.addItem(item);

  // Assert
  expect(cart.total).toBe(1200000);
});
```

### 3. 구현이 아닌 동작을 테스트한다

내부 구현 세부 사항에 의존하는 테스트는 리팩터링할 때마다 깨집니다. 외부에서 관찰 가능한 동작만 테스트해야 코드를 자유롭게 개선할 수 있습니다.

```javascript
// 나쁜 예: 내부 구현(프라이빗 배열)에 의존
test('상품이 올바른 위치에 저장된다', () => {
  cart.addItem(item);
  expect(cart._items[0]).toBe(item); // 내부 구조에 의존
});

// 좋은 예: 관찰 가능한 동작만 검증
test('추가한 상품이 장바구니에 포함된다', () => {
  cart.addItem(item);
  expect(cart.contains(item)).toBe(true);
});
```

### 4. 경계값과 엣지 케이스를 챙긴다

정상 케이스만 테스트하는 것은 절반만 하는 것입니다. 실제 버그는 경계에서 발생합니다.

검증해야 할 케이스 목록:

- 빈 입력, `null`, `undefined`
- 최솟값과 최댓값 경계
- 빈 배열, 단일 요소 배열
- 이미 존재하는 데이터로 다시 실행하는 경우
- 예상치 못한 타입의 입력

```javascript
describe('divide 함수', () => {
  test('두 숫자를 나눈다', () => {
    expect(divide(10, 2)).toBe(5);
  });

  test('0으로 나누면 에러를 던진다', () => {
    expect(() => divide(10, 0)).toThrow('0으로 나눌 수 없습니다');
  });

  test('음수를 나눌 수 있다', () => {
    expect(divide(-10, 2)).toBe(-5);
  });
});
```

### 5. 테스트는 독립적이어야 한다

각 테스트는 다른 테스트에 의존하지 않고 단독으로 실행될 수 있어야 합니다. 실행 순서가 달라지거나 특정 테스트만 실행해도 결과가 같아야 합니다.

`beforeEach`로 테스트마다 상태를 초기화하고, 전역 변수나 공유 상태를 최소화하세요.

```javascript
let cart;

beforeEach(() => {
  cart = new ShoppingCart(); // 매 테스트마다 새 인스턴스
});

test('빈 장바구니의 총액은 0이다', () => {
  expect(cart.total).toBe(0);
});

test('상품 추가 후 총액이 반영된다', () => {
  cart.addItem({ price: 5000 });
  expect(cart.total).toBe(5000);
});
```

## TDD: 테스트를 먼저 작성하는 개발 방법

**TDD(Test-Driven Development)** 는 코드를 짜기 전에 테스트를 먼저 작성하는 개발 방식입니다. 처음엔 낯설지만 익숙해지면 설계가 자연스럽게 개선되고 버그가 줄어드는 효과를 체감할 수 있습니다.

TDD의 3단계 사이클:

1. **Red**: 실패하는 테스트를 먼저 작성한다
2. **Green**: 테스트를 통과할 최소한의 코드를 작성한다
3. **Refactor**: 기능을 유지하면서 코드를 정리한다

TDD를 전체 프로젝트에 바로 적용하기 어렵다면, 새로운 버그를 수정할 때 "버그를 재현하는 테스트를 먼저 작성하고 나서 수정한다"는 원칙부터 시작해 보세요. 이것만으로도 같은 버그가 다시 발생하는 일을 막을 수 있습니다.

## 어디서부터 시작할까

기존 프로젝트에 테스트를 도입하기 막막하다면 이 순서를 추천합니다.

1. **자주 버그가 발생하는 함수**부터 테스트한다
2. **순수 함수**(같은 입력 → 같은 출력)부터 시작해 테스트 작성에 익숙해진다
3. 테스트 커버리지 목표를 너무 높게 잡지 않는다 — 70%도 충분한 시작점
4. **CI 파이프라인에 테스트를 연결**해 자동화한다

## 마치며

테스트 코드는 작성하는 시간보다 덕분에 절약되는 시간이 훨씬 많습니다. 리팩터링을 두려워하지 않게 되고, 새 기능 추가 후 기존 코드가 망가졌는지 걱정할 필요가 없어집니다. **테스트는 지금 당장 시간을 투자하고 나중에 자신감을 얻는 최고의 거래입니다.**
