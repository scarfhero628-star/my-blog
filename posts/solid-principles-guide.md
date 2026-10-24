---
title: SOLID 원칙 완전 가이드 — 유지보수하기 좋은 코드를 만드는 5가지 설계 원칙
date: 2026-10-24
tags: [SOLID, 객체지향, 설계 원칙, 코드 품질, 소프트웨어 설계]
description: 소프트웨어 설계의 핵심인 SOLID 5가지 원칙을 실전 예제와 함께 익혀 변경에 강하고 유지보수하기 좋은 코드를 작성하는 방법을 소개합니다.
---

## SOLID 원칙이란?

SOLID는 로버트 C. 마틴(Robert C. Martin, "Uncle Bob")이 정리한 객체지향 설계의 5가지 핵심 원칙입니다. 각 원칙의 앞글자를 따서 만든 약어로, 이 원칙들을 따르면 **변경에 강하고, 테스트하기 쉽고, 확장 가능한** 코드를 작성할 수 있습니다.

- **S**ingle Responsibility Principle — 단일 책임 원칙
- **O**pen/Closed Principle — 개방-폐쇄 원칙
- **L**iskov Substitution Principle — 리스코프 치환 원칙
- **I**nterface Segregation Principle — 인터페이스 분리 원칙
- **D**ependency Inversion Principle — 의존 역전 원칙

---

## 1. 단일 책임 원칙 (SRP)

> "클래스를 변경해야 하는 이유는 단 하나여야 한다."

하나의 클래스(혹은 함수)는 **하나의 책임**만 가져야 합니다. 여러 책임이 한 곳에 뭉쳐 있으면, 하나를 수정할 때 다른 기능에 예상치 못한 영향을 미칩니다.

### 나쁜 예

```typescript
class User {
  constructor(public name: string, public email: string) {}

  // 사용자 저장 — DB 책임
  save() {
    db.query(`INSERT INTO users ...`);
  }

  // 이메일 발송 — 이메일 서비스 책임
  sendWelcomeEmail() {
    emailService.send(this.email, "환영합니다!");
  }

  // HTML 렌더링 — UI 책임
  renderProfile() {
    return `<div>${this.name}</div>`;
  }
}
```

### 좋은 예

```typescript
class User {
  constructor(public name: string, public email: string) {}
}

class UserRepository {
  save(user: User) {
    db.query(`INSERT INTO users ...`);
  }
}

class UserEmailService {
  sendWelcome(user: User) {
    emailService.send(user.email, "환영합니다!");
  }
}
```

책임을 분리하면 DB 로직을 바꿔도 이메일 서비스 코드가 영향을 받지 않습니다.

---

## 2. 개방-폐쇄 원칙 (OCP)

> "소프트웨어 요소는 확장에는 열려 있고, 변경에는 닫혀 있어야 한다."

새 기능을 추가할 때 **기존 코드를 수정하지 않고** 새 코드를 추가하는 방식으로 확장해야 합니다. 이를 위해 추상화와 다형성을 적극 활용합니다.

### 나쁜 예

```typescript
function calculateDiscount(user: User, type: string): number {
  if (type === "vip") return user.price * 0.2;
  if (type === "member") return user.price * 0.1;
  // 새 할인 유형이 생길 때마다 이 함수를 수정해야 함
  return 0;
}
```

### 좋은 예

```typescript
interface DiscountStrategy {
  calculate(price: number): number;
}

class VipDiscount implements DiscountStrategy {
  calculate(price: number) { return price * 0.2; }
}

class MemberDiscount implements DiscountStrategy {
  calculate(price: number) { return price * 0.1; }
}

// 새 할인 정책은 새 클래스를 추가하기만 하면 됨
class SeasonDiscount implements DiscountStrategy {
  calculate(price: number) { return price * 0.15; }
}

function applyDiscount(price: number, strategy: DiscountStrategy): number {
  return price - strategy.calculate(price);
}
```

---

## 3. 리스코프 치환 원칙 (LSP)

> "자식 클래스는 언제나 부모 클래스로 교체 가능해야 한다."

부모 클래스를 사용하는 코드가 자식 클래스를 받았을 때도 **동일하게 동작**해야 합니다. 자식이 부모의 계약을 위반하면 예상치 못한 버그가 발생합니다.

### 나쁜 예 (고전적인 직사각형-정사각형 문제)

```typescript
class Rectangle {
  constructor(public width: number, public height: number) {}

  area() { return this.width * this.height; }
}

class Square extends Rectangle {
  // 정사각형은 가로·세로가 같아야 하므로 세터를 오버라이드
  set width(value: number) {
    this._width = value;
    this._height = value; // 부모의 계약을 어김!
  }
}

function testRectangle(r: Rectangle) {
  r.width = 4;
  r.height = 5;
  console.log(r.area()); // 20을 기대하지만 Square면 25가 나옴
}
```

### 해결 방법

`Square`가 `Rectangle`을 상속하는 대신, 공통 인터페이스(`Shape`)를 각자 구현하는 구조로 변경합니다.

```typescript
interface Shape {
  area(): number;
}

class Rectangle implements Shape {
  constructor(public width: number, public height: number) {}
  area() { return this.width * this.height; }
}

class Square implements Shape {
  constructor(public side: number) {}
  area() { return this.side * this.side; }
}
```

---

## 4. 인터페이스 분리 원칙 (ISP)

> "클라이언트는 자신이 사용하지 않는 메서드에 의존하지 않아야 한다."

하나의 큰 인터페이스보다 **목적에 맞는 작은 인터페이스 여러 개**가 낫습니다. 불필요한 메서드 구현을 강제하면 코드가 지저분해집니다.

### 나쁜 예

```typescript
interface Animal {
  eat(): void;
  fly(): void;  // 모든 동물이 날 수 있지는 않음
  swim(): void; // 모든 동물이 수영할 수 있지는 않음
}

class Dog implements Animal {
  eat() { /* ... */ }
  fly() { throw new Error("개는 날 수 없음"); } // 불필요한 구현
  swim() { /* ... */ }
}
```

### 좋은 예

```typescript
interface Eatable { eat(): void; }
interface Flyable  { fly(): void; }
interface Swimmable { swim(): void; }

class Dog implements Eatable, Swimmable {
  eat() { /* ... */ }
  swim() { /* ... */ }
}

class Bird implements Eatable, Flyable {
  eat() { /* ... */ }
  fly() { /* ... */ }
}
```

각 클래스는 실제로 필요한 인터페이스만 구현하면 됩니다.

---

## 5. 의존 역전 원칙 (DIP)

> "고수준 모듈은 저수준 모듈에 의존해서는 안 된다. 둘 다 추상화에 의존해야 한다."

비즈니스 로직(고수준)이 특정 구현체(저수준)에 직접 의존하면, 구현체를 바꿀 때 비즈니스 로직도 함께 수정해야 합니다. **인터페이스(추상화)를 사이에 두면** 구현체를 자유롭게 교체할 수 있습니다.

### 나쁜 예

```typescript
class OrderService {
  // 특정 구현체에 직접 의존
  private db = new MySQLDatabase();

  createOrder(order: Order) {
    this.db.save(order); // MySQL을 PostgreSQL로 바꾸면 여기도 수정 필요
  }
}
```

### 좋은 예

```typescript
// 추상화(인터페이스) 정의
interface Database {
  save(data: any): void;
}

// 구현체들
class MySQLDatabase implements Database {
  save(data: any) { /* MySQL 저장 */ }
}

class PostgresDatabase implements Database {
  save(data: any) { /* Postgres 저장 */ }
}

// 고수준 모듈은 인터페이스에만 의존
class OrderService {
  constructor(private db: Database) {}

  createOrder(order: Order) {
    this.db.save(order);
  }
}

// 사용 시 구현체를 주입 (의존성 주입)
const service = new OrderService(new MySQLDatabase());
```

테스트할 때는 `new MockDatabase()`를 주입하면 실제 DB 없이도 테스트할 수 있습니다.

---

## SOLID 원칙을 현실에서 적용하는 법

SOLID 원칙은 처음부터 완벽하게 적용하기보다 **점진적으로 적용**하는 게 현실적입니다.

### 실전 적용 팁

1. **새 기능을 추가할 때** OCP를 먼저 떠올리세요. 기존 코드를 수정하지 않고 추가할 방법이 있는지 확인합니다.
2. **클래스나 함수가 길어질 때** SRP를 점검하세요. "이 함수/클래스를 한 문장으로 설명할 수 있나?"를 물어보면 됩니다.
3. **테스트 코드 작성이 어려울 때** DIP를 확인하세요. 의존성이 직접 연결되어 있으면 테스트가 어렵습니다.
4. **완벽한 SOLID보다 실용적인 코드**가 더 중요합니다. 과도한 추상화는 오히려 코드를 복잡하게 만듭니다.

### 원칙 간 관계

SOLID 원칙은 서로 독립적이지 않습니다. DIP를 잘 지키면 OCP가 자연스럽게 따라오고, SRP를 지키면 ISP를 위반하기 어렵습니다. **하나를 잘 적용하면 다른 원칙도 함께 개선**되는 경우가 많습니다.

---

## 정리

| 원칙 | 한 줄 요약 | 핵심 키워드 |
|------|-----------|------------|
| SRP | 하나의 클래스, 하나의 이유 | 책임 분리 |
| OCP | 기존 코드 건드리지 않고 확장 | 추상화 |
| LSP | 자식은 부모를 완전히 대체 가능 | 다형성 |
| ISP | 필요한 인터페이스만 구현 | 인터페이스 분리 |
| DIP | 구현이 아닌 추상화에 의존 | 의존성 주입 |

SOLID 원칙은 "좋은 코드란 무엇인가"에 대한 수십 년의 경험이 응축된 지혜입니다. 모든 규칙을 교조적으로 따르기보다, **왜 이 원칙이 필요한지** 이해하고 상황에 맞게 적용하는 것이 핵심입니다. 처음에는 어색하더라도, 꾸준히 연습하다 보면 자연스럽게 더 나은 설계를 떠올리게 됩니다.
