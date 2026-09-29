---
title: SQL 쿼리 최적화 실전 가이드 — 느린 쿼리를 10배 빠르게 만드는 방법
date: 2026-09-29
tags: [SQL, 데이터베이스, 쿼리 최적화, 백엔드, 성능]
description: 실무에서 자주 마주치는 느린 쿼리의 원인을 분석하고, 인덱스 활용부터 쿼리 재작성까지 즉시 적용할 수 있는 SQL 최적화 기법을 소개합니다.
---

백엔드 개발을 하다 보면 어느 순간 API 응답이 갑자기 느려지는 경험을 하게 됩니다. 원인을 추적해 보면 대부분 데이터베이스 쿼리 문제입니다. 테이블에 데이터가 몇 천 건일 때는 전혀 문제없던 쿼리가 수백만 건이 쌓이면 수 초씩 걸리기도 합니다. 이 글에서는 실무에서 바로 적용할 수 있는 SQL 쿼리 최적화 기법을 정리합니다.

## EXPLAIN으로 쿼리 실행 계획 읽기

느린 쿼리를 고치기 전에 먼저 **왜 느린지** 파악해야 합니다. `EXPLAIN` 키워드는 데이터베이스가 쿼리를 어떻게 실행하는지 보여줍니다.

```sql
EXPLAIN SELECT * FROM orders
WHERE user_id = 123
ORDER BY created_at DESC;
```

실행 결과에서 확인해야 할 핵심 항목은 다음과 같습니다.

- **type**: `ALL`이면 풀 테이블 스캔, `ref`/`range`/`const`이면 인덱스 활용
- **rows**: 예상 검색 행 수. 숫자가 클수록 위험 신호
- **Extra**: `Using filesort`, `Using temporary`가 보이면 성능 저하 가능성

MySQL에서는 `EXPLAIN ANALYZE`를 사용하면 실제 실행 통계도 함께 볼 수 있습니다.

## 인덱스를 제대로 활용하기

인덱스는 SQL 최적화의 핵심입니다. 올바르게 사용하면 수백 배의 성능 향상을 기대할 수 있습니다.

### 언제 인덱스를 만들어야 하나?

- `WHERE` 절에서 자주 조건으로 사용하는 컬럼
- `JOIN`에서 연결 키로 사용하는 컬럼
- `ORDER BY`, `GROUP BY`에 자주 등장하는 컬럼

```sql
-- user_id로 자주 검색한다면 인덱스 추가
CREATE INDEX idx_orders_user_id ON orders(user_id);

-- 복합 인덱스: 자주 함께 조회하는 컬럼
CREATE INDEX idx_orders_user_date ON orders(user_id, created_at);
```

### 인덱스를 무력화하는 실수

인덱스가 있어도 아래 패턴에서는 인덱스를 사용하지 못합니다.

```sql
-- 함수로 감싸면 인덱스 무효
WHERE YEAR(created_at) = 2024  -- 나쁜 예
WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01'  -- 좋은 예

-- LIKE 앞에 와일드카드를 쓰면 인덱스 무효
WHERE name LIKE '%홍길동%'  -- 나쁜 예
WHERE name LIKE '홍길동%'   -- 좋은 예 (앞 와일드카드 없음)
```

## N+1 문제 해결하기

N+1 문제는 ORM을 사용할 때 가장 빈번하게 발생하는 성능 이슈입니다. 목록 1개를 가져오는 쿼리 1번 + 각 항목의 연관 데이터를 가져오는 쿼리 N번이 실행되는 구조입니다.

### 문제가 있는 코드 예시 (Pseudo ORM)

```javascript
// 주문 100건 조회 후 각각 유저 정보 별도 쿼리 → 총 101번 쿼리
const orders = await Order.findAll();
for (const order of orders) {
  const user = await User.findById(order.userId); // N번 추가 쿼리
}
```

### JOIN으로 한 번에 해결

```sql
-- 1번의 쿼리로 주문 + 유저 정보를 함께 조회
SELECT o.id, o.total_price, u.name, u.email
FROM orders o
INNER JOIN users u ON o.user_id = u.id
WHERE o.status = 'completed';
```

ORM을 쓴다면 `eager loading` 또는 `include` 옵션을 활용해 JOIN 쿼리를 유도하는 것이 좋습니다.

## SELECT * 대신 필요한 컬럼만 조회하기

`SELECT *`는 편리하지만 불필요한 데이터를 모두 전송합니다. 데이터 양이 많아지면 네트워크 전송 비용과 메모리 사용량이 늘어납니다.

```sql
-- 나쁜 예: 모든 컬럼 조회
SELECT * FROM products WHERE category = 'electronics';

-- 좋은 예: 필요한 컬럼만 조회
SELECT id, name, price, stock FROM products WHERE category = 'electronics';
```

특히 `TEXT`나 `BLOB` 같은 대용량 컬럼이 포함된 테이블에서는 이 차이가 매우 큽니다.

## 페이지네이션 쿼리 최적화

대용량 테이블에서 `OFFSET`이 큰 페이지네이션은 성능 문제를 일으킵니다. `OFFSET 10000`이면 데이터베이스는 앞의 1만 건을 읽은 뒤 버리는 방식으로 동작합니다.

```sql
-- 느린 방식: OFFSET이 클수록 점점 느려짐
SELECT * FROM posts ORDER BY id DESC LIMIT 20 OFFSET 10000;

-- 빠른 방식: 커서 기반 페이지네이션
SELECT * FROM posts
WHERE id < :last_id  -- 이전 페이지의 마지막 id
ORDER BY id DESC
LIMIT 20;
```

커서 기반 페이지네이션은 항상 동일한 성능을 유지하며, 무한 스크롤 UI와도 잘 어울립니다.

## 자주 발생하는 실수 체크리스트

쿼리를 작성하거나 리뷰할 때 아래 항목을 확인해 보세요.

- [ ] WHERE 조건 컬럼에 인덱스가 있는가?
- [ ] 인덱스를 무력화하는 함수나 묵시적 형변환이 없는가?
- [ ] N+1 문제가 발생하지 않는가?
- [ ] `SELECT *` 대신 필요한 컬럼만 조회하는가?
- [ ] `OFFSET` 기반 페이지네이션을 대규모 테이블에 사용하지 않는가?
- [ ] 서브쿼리를 JOIN으로 대체할 수 있는가?

## 마치며

SQL 최적화는 한 번에 완성되지 않습니다. 데이터가 늘어날수록 기존 쿼리도 계속 점검해야 합니다. `EXPLAIN`으로 실행 계획을 읽는 습관을 들이고, 느린 쿼리 로그(Slow Query Log)를 활성화해 주기적으로 모니터링하는 것이 중요합니다. 작은 최적화 하나가 서버 비용과 사용자 경험을 크게 개선할 수 있습니다.
