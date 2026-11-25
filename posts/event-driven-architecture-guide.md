---
title: 이벤트 주도 아키텍처(EDA) 완전 가이드 — 느슨한 결합으로 확장 가능한 시스템 만들기
date: 2026-11-25
tags: [이벤트 주도 아키텍처, EDA, 메시지 큐, Kafka, 아키텍처]
description: 이벤트 주도 아키텍처(EDA)의 핵심 개념과 패턴을 이해하고, 메시지 브로커를 활용해 느슨하게 결합된 확장 가능한 시스템을 설계하는 방법을 소개합니다.
---

## 이벤트 주도 아키텍처란?

이벤트 주도 아키텍처(Event-Driven Architecture, EDA)는 시스템 구성 요소들이 **이벤트(사건)**를 통해 통신하는 소프트웨어 설계 패러다임입니다. 각 컴포넌트는 서로를 직접 호출하는 대신, 이벤트를 발행(publish)하거나 구독(subscribe)하는 방식으로 느슨하게 연결됩니다.

예를 들어 전자상거래 시스템에서 "주문 완료" 이벤트가 발생하면:

- 결제 서비스가 결제를 처리하고
- 재고 서비스가 재고를 감소시키며
- 알림 서비스가 주문 확인 이메일을 발송합니다

이 모든 처리가 서로를 직접 알 필요 없이 완전히 독립적으로 이루어집니다. 한 서비스가 추가되거나 변경되어도 다른 서비스의 코드는 전혀 수정할 필요가 없습니다.

## 핵심 구성 요소

### 이벤트 생산자 (Event Producer)

이벤트를 생성하고 발행하는 컴포넌트입니다. 비즈니스 로직이 발생할 때 이벤트를 메시지 브로커에 전달합니다. 생산자는 누가 이 이벤트를 소비하는지 알 필요가 없습니다.

### 이벤트 소비자 (Event Consumer)

특정 이벤트를 구독하고 처리하는 컴포넌트입니다. 하나의 이벤트를 여러 소비자가 독립적으로 처리할 수 있으며, 각 소비자는 자신의 관심 이벤트만 구독합니다.

### 메시지 브로커 (Message Broker)

이벤트를 중계하는 미들웨어입니다. 생산자와 소비자 사이에서 이벤트를 안정적으로 전달하고, 필요하다면 버퍼 역할도 합니다. 대표적으로 **Apache Kafka**, **RabbitMQ**, **AWS SQS/SNS** 등이 있습니다.

## 주요 패턴

### 1. Pub/Sub (발행-구독)

가장 기본적인 EDA 패턴입니다. 생산자는 토픽(채널)에 이벤트를 발행하고, 해당 토픽을 구독한 모든 소비자가 이벤트를 받습니다.

```javascript
// Node.js EventEmitter를 활용한 간단한 Pub/Sub 예시
const EventEmitter = require('events');
const emitter = new EventEmitter();

// 각 서비스가 독립적으로 이벤트를 구독
emitter.on('order:created', (order) => {
  console.log('결제 처리 시작:', order.id);
});

emitter.on('order:created', (order) => {
  console.log('재고 감소 처리:', order.items);
});

emitter.on('order:created', (order) => {
  console.log('확인 이메일 발송:', order.customerEmail);
});

// 주문 서비스에서 이벤트 발행 — 소비자를 전혀 알지 못함
emitter.emit('order:created', {
  id: 'ORD-001',
  items: ['상품A'],
  customerEmail: 'user@example.com',
});
```

### 2. 이벤트 소싱 (Event Sourcing)

상태를 직접 저장하는 대신, 상태 변경 **이벤트를 순서대로 저장**하는 패턴입니다. 모든 변경 이력이 보존되어 감사 추적과 임의 시점으로의 복구가 가능합니다.

```javascript
// 장바구니 이벤트 목록 (데이터베이스에 이렇게 저장됨)
const cartEvents = [
  { type: 'CartCreated', cartId: 'CART-001', timestamp: '2026-11-25T09:00:00Z' },
  { type: 'ItemAdded', cartId: 'CART-001', item: '상품A', price: 10000 },
  { type: 'ItemAdded', cartId: 'CART-001', item: '상품B', price: 20000 },
  { type: 'ItemRemoved', cartId: 'CART-001', item: '상품A' },
];

// 이벤트를 순서대로 재생(replay)해 현재 상태를 복원
function replayCart(events) {
  return events.reduce((cart, event) => {
    switch (event.type) {
      case 'CartCreated':
        return { id: event.cartId, items: [], total: 0 };
      case 'ItemAdded':
        return {
          ...cart,
          items: [...cart.items, event.item],
          total: cart.total + event.price,
        };
      case 'ItemRemoved':
        return {
          ...cart,
          items: cart.items.filter((i) => i !== event.item),
        };
      default:
        return cart;
    }
  }, {});
}

// 현재 장바구니 상태: { id: 'CART-001', items: ['상품B'], total: 20000 }
const currentCart = replayCart(cartEvents);
```

### 3. CQRS (Command Query Responsibility Segregation)

읽기(Query)와 쓰기(Command) 책임을 분리하는 패턴입니다. EDA와 함께 사용하면 쓰기 이벤트를 처리해 별도의 읽기 모델을 유지하고, 각 모델을 독립적으로 최적화할 수 있습니다.

## Apache Kafka 실전 활용

Kafka는 고처리량 분산 이벤트 스트리밍 플랫폼으로 대규모 EDA의 핵심 도구입니다.

```javascript
const { Kafka } = require('kafkajs');

const kafka = new Kafka({ brokers: ['localhost:9092'] });

// 이벤트 발행 (주문 서비스)
async function publishOrderEvent(order) {
  const producer = kafka.producer();
  await producer.connect();
  await producer.send({
    topic: 'orders',
    messages: [{ key: order.id, value: JSON.stringify(order) }],
  });
  await producer.disconnect();
}

// 이벤트 소비 (결제 서비스)
async function startPaymentConsumer() {
  const consumer = kafka.consumer({ groupId: 'payment-service' });
  await consumer.connect();
  await consumer.subscribe({ topic: 'orders', fromBeginning: false });

  await consumer.run({
    eachMessage: async ({ topic, partition, message }) => {
      const order = JSON.parse(message.value.toString());
      await processPayment(order);
    },
  });
}
```

## EDA의 장단점

### 장점

- **느슨한 결합**: 서비스들이 서로를 직접 알 필요가 없어 독립적으로 개발·배포·확장 가능
- **높은 확장성**: 소비자를 추가하기만 하면 처리 용량을 쉽게 늘릴 수 있음
- **유연성**: 기존 코드를 수정하지 않고 이벤트 구독만으로 새로운 기능 추가 가능
- **장애 격리**: 소비자 서비스가 일시 장애가 나도 이벤트가 큐에 보관되어 복구 후 처리 가능
- **완벽한 감사 로그**: 이벤트 소싱과 함께 사용하면 모든 상태 변경 이력을 보존

### 단점

- **디버깅의 어려움**: 비동기 흐름은 추적이 복잡해 Observability 도구(분산 트레이싱)가 필수
- **순서 보장의 복잡성**: 이벤트 순서가 중요한 경우 파티셔닝 등 추가 설계가 필요
- **중복 처리 위험**: 네트워크 오류 시 이벤트가 재전송될 수 있어 멱등성(idempotency) 설계가 필요
- **높은 초기 복잡도**: 메시지 브로커 운영과 이벤트 스키마 관리가 추가되어 작은 프로젝트엔 과할 수 있음

## 언제 EDA를 도입해야 할까?

**EDA가 적합한 상황:**

- 여러 서비스가 하나의 비즈니스 이벤트에 반응해야 할 때
- 실시간 데이터 처리나 스트리밍이 필요할 때
- 마이크로서비스 간 결합도를 낮춰야 할 때
- 감사 로그나 이벤트 이력이 비즈니스적으로 중요할 때

**EDA가 과할 수 있는 상황:**

- 단순한 CRUD 애플리케이션이나 소규모 모놀리식 서비스
- 팀 규모가 작고 서비스 수가 적을 때
- 즉각적인 동기 응답이 반드시 필요한 경우

## 마치며

이벤트 주도 아키텍처는 현대 분산 시스템을 느슨하게 결합하고 유연하게 확장하는 강력한 패러다임입니다. 단, 추가되는 복잡도를 반드시 고려해야 합니다. 작은 서비스라면 Node.js `EventEmitter`나 간단한 Pub/Sub부터 시작해 점진적으로 Kafka 같은 본격적인 메시지 브로커로 확장하는 전략이 현명합니다. 중요한 것은 비즈니스 도메인의 이벤트를 명확히 정의하는 것에서 시작하는 것입니다.
