---
title: Redis 캐싱 전략 완전 가이드 — 서버 성능을 10배 높이는 핵심 패턴
date: 2026-10-16
tags: [Redis, 캐싱, 백엔드, 성능, Node.js]
description: 실무에서 바로 적용할 수 있는 Redis 캐싱 패턴과 전략을 통해 데이터베이스 부하를 줄이고 서버 응답 속도를 획기적으로 개선하는 방법을 소개합니다.
---

## Redis가 왜 필요한가?

웹 애플리케이션이 성장할수록 데이터베이스 쿼리는 병목이 됩니다. 같은 상품 정보를 초당 수백 번 조회하거나, 복잡한 집계 쿼리를 반복 실행하면 DB는 금방 한계에 도달합니다.

**Redis(Remote Dictionary Server)**는 메모리 기반의 Key-Value 저장소입니다. 데이터를 RAM에 보관하기 때문에 디스크 기반 데이터베이스보다 읽기 속도가 100~1000배 빠릅니다. 캐시, 세션 저장, 실시간 순위 집계 등 다양한 용도로 활용됩니다.

---

## 핵심 캐싱 패턴 3가지

### 1. Cache-Aside (지연 로딩)

가장 범용적인 패턴입니다. 캐시에 데이터가 없을 때만 DB를 조회하고, 그 결과를 캐시에 저장합니다.

```
요청 → 캐시 확인 → (히트) 바로 반환
                  → (미스) DB 조회 → 캐시 저장 → 반환
```

```javascript
// Node.js + ioredis 예시
async function getProduct(productId) {
  const cacheKey = `product:${productId}`;

  // 1. 캐시 확인
  const cached = await redis.get(cacheKey);
  if (cached) {
    return JSON.parse(cached);
  }

  // 2. DB 조회
  const product = await db.products.findById(productId);
  if (!product) return null;

  // 3. 캐시에 저장 (TTL: 1시간)
  await redis.setex(cacheKey, 3600, JSON.stringify(product));

  return product;
}
```

**장점:** 캐시 장애가 발생해도 DB로 fallback됩니다.
**단점:** 캐시 미스 시 DB 조회 비용이 발생하며, 처음 요청은 항상 느립니다.

---

### 2. Write-Through (쓰기 투과)

데이터를 쓸 때 캐시와 DB를 동시에 업데이트합니다. 데이터 일관성이 높지만, 쓰기 지연이 생길 수 있습니다.

```javascript
async function updateProduct(productId, data) {
  const cacheKey = `product:${productId}`;

  // DB와 캐시를 동시에 업데이트
  await Promise.all([
    db.products.update(productId, data),
    redis.setex(cacheKey, 3600, JSON.stringify(data)),
  ]);
}
```

**장점:** 캐시와 DB가 항상 최신 상태를 유지합니다.
**단점:** 쓰기 성능이 떨어지고, 자주 읽히지 않는 데이터도 캐시에 저장됩니다.

---

### 3. Write-Back (쓰기 지연)

데이터를 캐시에 먼저 쓰고, 주기적으로 DB에 반영합니다. 쓰기가 빈번한 환경에서 강력합니다.

```
쓰기 요청 → 캐시만 업데이트 → (비동기) 일괄 DB 저장
```

**장점:** 쓰기 성능이 매우 높습니다.
**단점:** 캐시 장애 시 데이터 유실 위험이 있습니다.

---

## TTL(만료 시간) 설정 전략

TTL을 잘못 설정하면 오래된 데이터를 서비스하거나 캐시 효율이 떨어집니다.

| 데이터 유형 | 권장 TTL | 이유 |
|---|---|---|
| 상품 정보 | 1~12시간 | 자주 변경되지 않음 |
| 사용자 세션 | 30분~2시간 | 활성 세션 유지 |
| 뉴스·공지 | 5~30분 | 실시간성이 중요 |
| 환율·주가 | 1~5분 | 잦은 변경 |
| 통계·집계 | 10분~1시간 | 계산 비용이 높음 |

```javascript
// TTL을 환경별로 관리
const TTL = {
  product: 3600,      // 1시간
  userSession: 7200,  // 2시간
  searchResult: 300,  // 5분
};

await redis.setex(`product:${id}`, TTL.product, JSON.stringify(data));
```

---

## 캐시 키 네이밍 규칙

명확한 키 네이밍은 유지보수의 핵심입니다. 아래 패턴을 권장합니다.

```
{서비스}:{리소스}:{ID}:{변형}
```

```javascript
// 좋은 예시
'shop:product:42'
'shop:user:1234:profile'
'shop:category:electronics:list'
'api:search:query:iphone:page:1'

// 나쁜 예시
'p42'        // 의미 불명확
'user1234'   // 서비스 구분 없음
'list'       // 너무 광범위
```

### 네임스페이스로 일괄 삭제

관련 캐시를 묶어 삭제할 때 유용합니다.

```javascript
// 상품 관련 모든 캐시 삭제
async function invalidateProductCache(productId) {
  const keys = await redis.keys(`shop:product:${productId}*`);
  if (keys.length > 0) {
    await redis.del(...keys);
  }
}
```

> **주의:** `KEYS` 명령어는 프로덕션에서 성능 문제를 일으킬 수 있습니다. 데이터가 많다면 `SCAN` 명령어를 사용하세요.

---

## Cache Stampede 방지

캐시가 만료되는 순간 동시에 수백 개의 요청이 DB로 몰리는 현상을 **Cache Stampede(캐시 스탬피드)**라고 합니다.

### 해결 방법 1: 뮤텍스 잠금

```javascript
async function getProductSafe(productId) {
  const cacheKey = `product:${productId}`;
  const lockKey = `lock:product:${productId}`;

  const cached = await redis.get(cacheKey);
  if (cached) return JSON.parse(cached);

  // 락 획득 시도 (최대 5초 대기)
  const lock = await redis.set(lockKey, '1', 'EX', 5, 'NX');

  if (lock) {
    // 락 획득 성공 → DB 조회
    const product = await db.products.findById(productId);
    await redis.setex(cacheKey, 3600, JSON.stringify(product));
    await redis.del(lockKey);
    return product;
  } else {
    // 락 획득 실패 → 잠시 대기 후 재시도
    await new Promise(resolve => setTimeout(resolve, 100));
    return getProductSafe(productId);
  }
}
```

### 해결 방법 2: 확률적 조기 갱신 (Probabilistic Early Expiration)

TTL이 얼마 남지 않았을 때 일부 요청만 먼저 캐시를 갱신합니다. 간단하면서 효과적입니다.

```javascript
async function getWithEarlyExpiry(key, fetchFn, ttl) {
  const data = await redis.get(key);
  if (!data) {
    const fresh = await fetchFn();
    await redis.setex(key, ttl, JSON.stringify(fresh));
    return fresh;
  }

  const remainingTtl = await redis.ttl(key);
  // TTL의 20% 미만이면 10% 확률로 조기 갱신
  if (remainingTtl < ttl * 0.2 && Math.random() < 0.1) {
    fetchFn().then(fresh => redis.setex(key, ttl, JSON.stringify(fresh)));
  }

  return JSON.parse(data);
}
```

---

## Redis 데이터 구조 활용

Redis는 단순 Key-Value 외에도 다양한 자료구조를 지원합니다.

### Sorted Set으로 실시간 랭킹

```javascript
// 점수 업데이트
await redis.zadd('ranking:weekly', score, userId);

// 상위 10명 조회
const top10 = await redis.zrevrange('ranking:weekly', 0, 9, 'WITHSCORES');
```

### Hash로 객체 일부 필드만 업데이트

```javascript
// 전체 JSON 대신 필드 단위로 저장
await redis.hset(`user:${userId}`, 'name', '홍길동', 'score', 1500);

// 특정 필드만 읽기 (전체 로드 불필요)
const name = await redis.hget(`user:${userId}`, 'name');
```

---

## 실무에서 자주 하는 실수

1. **TTL 없이 저장** — 메모리가 끝없이 증가합니다. 항상 TTL을 설정하세요.
2. **너무 큰 값 저장** — 1MB 이상의 JSON을 캐시하면 오히려 느려집니다. 필요한 필드만 저장하세요.
3. **캐시 일관성 방치** — DB를 업데이트할 때 캐시 무효화(invalidation)를 잊으면 오래된 데이터가 서비스됩니다.
4. **직렬화 오버헤드 무시** — JSON.parse/stringify는 비용이 있습니다. 빈번한 객체는 MessagePack 같은 이진 형식을 고려하세요.

---

## 마치며

Redis 캐싱은 복잡한 인프라 없이도 서버 성능을 극적으로 향상시킬 수 있는 가장 효과적인 방법 중 하나입니다. Cache-Aside로 시작해 필요에 따라 Write-Through나 Write-Back을 도입하고, TTL과 키 네이밍 전략을 체계적으로 관리하면 대부분의 성능 문제를 해결할 수 있습니다.

작은 서비스부터 캐시 레이어를 도입하는 습관을 들이면, 트래픽이 급증할 때 훨씬 여유롭게 대응할 수 있습니다.
