---
title: CQRS와 이벤트 소싱 완전 가이드 — 읽기·쓰기 분리로 확장성 높은 시스템 설계하기
date: 2026-11-28
tags: [CQRS, 이벤트 소싱, 아키텍처, 백엔드, 시스템 설계]
description: 명령과 조회를 분리하는 CQRS 패턴과 상태 변경 이력을 이벤트로 저장하는 이벤트 소싱을 함께 적용해 확장성 있고 감사 가능한 시스템을 설계하는 방법을 소개합니다.
---

## CQRS란 무엇인가?

**CQRS(Command Query Responsibility Segregation)**는 시스템의 **쓰기(Command)** 와 **읽기(Query)** 책임을 명확히 분리하는 아키텍처 패턴입니다. Greg Young이 제안한 이 패턴은 단순해 보이지만, 대규모 시스템에서 발생하는 복잡성을 근본적으로 줄여줍니다.

전통적인 CRUD 방식은 같은 모델로 읽기와 쓰기를 모두 처리합니다. 이는 작은 시스템에서는 문제가 없지만, 규모가 커질수록 다음과 같은 문제가 생깁니다.

- 읽기에 최적화된 구조와 쓰기에 최적화된 구조가 충돌함
- 복잡한 비즈니스 규칙 검증이 조회 성능에 영향을 미침
- 읽기·쓰기 부하를 독립적으로 확장하기 어려움

CQRS는 이 두 흐름을 완전히 분리해 각각 최적화할 수 있게 해줍니다.

## Command와 Query 모델 분리하기

### Command 모델 (쓰기)

Command 모델은 비즈니스 규칙 검증과 상태 변경에 집중합니다. 도메인 로직이 풍부하게 담겨 있어야 합니다.

```typescript
// Command: 주문 생성
class CreateOrderCommand {
  constructor(
    public readonly userId: string,
    public readonly items: OrderItem[],
    public readonly shippingAddress: Address,
  ) {}
}

// Command Handler
class CreateOrderHandler {
  async handle(command: CreateOrderCommand): Promise<string> {
    const user = await this.userRepo.findById(command.userId);
    if (!user.canOrder()) throw new Error('주문 불가 상태입니다.');

    const order = Order.create(command.userId, command.items, command.shippingAddress);
    await this.orderRepo.save(order);
    return order.id;
  }
}
```

### Query 모델 (읽기)

Query 모델은 화면에 필요한 데이터를 최적의 형태로 직접 반환합니다. ORM이나 도메인 객체를 거칠 필요 없이 쿼리 결과를 DTO로 바로 매핑합니다.

```typescript
// Query: 주문 목록 조회
class GetOrderListQuery {
  constructor(
    public readonly userId: string,
    public readonly page: number,
  ) {}
}

// Query Handler — 복잡한 JOIN도 그냥 쿼리로
class GetOrderListHandler {
  async handle(query: GetOrderListQuery): Promise<OrderListDto[]> {
    return this.db.raw(`
      SELECT o.id, o.status, o.total_price, o.created_at,
             COUNT(oi.id) as item_count
      FROM orders o
      JOIN order_items oi ON o.id = oi.order_id
      WHERE o.user_id = ?
      GROUP BY o.id
      ORDER BY o.created_at DESC
      LIMIT 20 OFFSET ?
    `, [query.userId, query.page * 20]);
  }
}
```

읽기 모델은 별도의 데이터베이스(예: Redis, Elasticsearch, 읽기 전용 복제본)를 사용할 수 있어 조회 성능을 독립적으로 최적화할 수 있습니다.

## 이벤트 소싱(Event Sourcing)이란?

이벤트 소싱은 현재 상태를 저장하는 대신 **상태 변경을 일으킨 이벤트 목록**을 저장하는 패턴입니다. 현재 상태는 처음부터 이벤트를 차례로 적용(replay)해 도출합니다.

### 왜 이벤트 소싱인가?

| 전통적 방식 | 이벤트 소싱 |
|---|---|
| 현재 상태만 저장 | 모든 변경 이력 보존 |
| 과거 상태 조회 불가 | 특정 시점 상태 복원 가능 |
| 감사 로그 별도 필요 | 이벤트 자체가 감사 로그 |
| 디버깅 어려움 | 이벤트 재생으로 버그 재현 |

### 이벤트 소싱 구현 예시

```typescript
// 도메인 이벤트 정의
interface OrderCreated {
  type: 'ORDER_CREATED';
  orderId: string;
  userId: string;
  items: OrderItem[];
  createdAt: Date;
}

interface OrderShipped {
  type: 'ORDER_SHIPPED';
  orderId: string;
  trackingNumber: string;
  shippedAt: Date;
}

type OrderEvent = OrderCreated | OrderShipped | OrderCancelled;

// 이벤트로부터 상태 복원
class Order {
  id: string;
  status: 'PENDING' | 'SHIPPED' | 'CANCELLED' = 'PENDING';

  static fromEvents(events: OrderEvent[]): Order {
    const order = new Order();
    for (const event of events) {
      order.apply(event);
    }
    return order;
  }

  private apply(event: OrderEvent) {
    switch (event.type) {
      case 'ORDER_CREATED':
        this.id = event.orderId;
        this.status = 'PENDING';
        break;
      case 'ORDER_SHIPPED':
        this.status = 'SHIPPED';
        break;
      case 'ORDER_CANCELLED':
        this.status = 'CANCELLED';
        break;
    }
  }
}
```

## CQRS + 이벤트 소싱 조합하기

두 패턴은 함께 사용할 때 시너지를 발휘합니다. 쓰기 모델에서 이벤트를 발행하면, 이를 구독한 읽기 모델이 자신의 뷰를 업데이트하는 방식입니다.

```
[사용자 요청]
    │
    ▼
[Command Handler] → 도메인 이벤트 발행
    │
    ▼
[Event Store] ← 이벤트 영속 저장
    │
    ▼
[Event Handler] → 읽기 모델(View) 업데이트
    │
    ▼
[Query Handler] ← 최적화된 읽기 뷰 조회
```

## 도입 시 주의할 점

CQRS와 이벤트 소싱은 강력하지만, 모든 시스템에 적합하지는 않습니다.

### 적합한 경우
- 읽기와 쓰기 부하가 현저히 다른 시스템
- 완벽한 감사 로그가 필요한 금융·의료 도메인
- 복잡한 비즈니스 규칙이 있는 도메인 주도 설계(DDD) 프로젝트

### 주의해야 할 경우
- 단순 CRUD 위주의 소규모 프로젝트 — 오버엔지니어링이 될 수 있음
- **최종 일관성(Eventual Consistency)** 을 허용하지 않는 요구사항 — 이벤트 전파 사이에 짧은 지연이 발생할 수 있음
- 팀에 이 패턴을 이해하는 인원이 없는 경우 — 학습 비용이 상당함

## 마치며

CQRS와 이벤트 소싱은 복잡한 도메인을 명확하게 모델링하고, 시스템의 과거를 완전히 추적할 수 있게 해주는 강력한 도구입니다. 처음에는 보일러플레이트 코드가 많아 보이지만, 시스템이 성장할수록 그 가치가 드러납니다. 작은 바운디드 컨텍스트 하나에 먼저 적용해 팀이 패턴에 익숙해지는 접근법을 추천합니다.
