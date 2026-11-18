---
title: 데이터베이스 설계 완전 가이드 — ERD와 정규화로 견고한 스키마 만들기
date: 2026-11-18
tags: [데이터베이스, ERD, 정규화, SQL, 백엔드]
description: 좋은 데이터베이스 스키마 설계를 위한 ERD 작성법과 정규화 원칙, 실무 패턴을 단계별로 소개합니다.
---

## 왜 데이터베이스 설계가 중요한가

애플리케이션의 성능과 유지보수성은 코드만큼이나 **데이터베이스 스키마 설계**에 달려 있습니다. 잘못 설계된 스키마는 시간이 지날수록 쿼리 성능을 떨어뜨리고, 중복 데이터를 양산하며, 비즈니스 요구 변경에 취약한 구조를 만듭니다.

반대로 처음부터 견고하게 설계된 스키마는 복잡한 쿼리도 간결하게 작성하게 해주고, 데이터 무결성을 자동으로 보장하며, 서비스가 성장해도 흔들리지 않는 기반이 됩니다.

이 가이드에서는 ERD 작성법부터 정규화 원칙, 실전에서 자주 쓰는 설계 패턴까지 단계별로 살펴봅니다.

---

## ERD란 무엇인가

**ERD(Entity-Relationship Diagram)**는 데이터베이스의 구조를 시각적으로 표현하는 다이어그램입니다. 테이블(엔티티)과 그 사이의 관계를 한눈에 파악할 수 있어 설계 초기 단계에서 협업과 검토에 필수적입니다.

### 핵심 구성 요소

- **엔티티(Entity)**: 테이블에 해당하는 개념적 객체 (예: `User`, `Order`, `Product`)
- **속성(Attribute)**: 엔티티가 가지는 특성 (예: `name`, `email`, `created_at`)
- **관계(Relationship)**: 엔티티 사이의 연결 (예: 사용자는 여러 주문을 가짐)
- **카디널리티(Cardinality)**: 관계의 수적 특성 (1:1, 1:N, N:M)

### 카디널리티 유형

| 관계 유형 | 예시 |
|-----------|------|
| 1:1 | 사용자 — 프로필 |
| 1:N | 사용자 — 주문 (한 사용자가 여러 주문) |
| N:M | 주문 — 상품 (한 주문에 여러 상품, 한 상품이 여러 주문에) |

N:M 관계는 반드시 **중간 테이블(Junction Table)**로 풀어야 합니다.

```sql
-- N:M 관계를 중간 테이블로 해소
CREATE TABLE order_items (
  order_id   BIGINT        NOT NULL REFERENCES orders(id),
  product_id BIGINT        NOT NULL REFERENCES products(id),
  quantity   INT           NOT NULL DEFAULT 1,
  price      DECIMAL(10,2) NOT NULL,
  PRIMARY KEY (order_id, product_id)
);
```

---

## 정규화 — 중복 없이 일관된 데이터

**정규화(Normalization)**는 데이터 중복을 최소화하고 갱신 이상(Update Anomaly)을 방지하기 위해 테이블을 단계적으로 분리하는 과정입니다.

### 제1정규형 (1NF)

한 셀에 하나의 값만 저장합니다. 반복 그룹을 없애고 각 행을 고유하게 식별할 수 있어야 합니다.

```sql
-- 위반 예시: tags 컬럼에 여러 값 저장
-- user_id | tags
-- 1       | 'javascript,css,html'

-- 1NF 준수: 별도 테이블로 분리
CREATE TABLE user_tags (
  user_id BIGINT      NOT NULL REFERENCES users(id),
  tag     VARCHAR(50) NOT NULL,
  PRIMARY KEY (user_id, tag)
);
```

### 제2정규형 (2NF)

1NF를 만족하면서, **부분 함수 종속**을 없앱니다. 복합 기본키에서 일부 키에만 의존하는 속성을 별도 테이블로 분리합니다.

```sql
-- 위반 예시: product_name이 product_id에만 종속 (복합키 중 일부)
-- order_items: order_id, product_id, quantity, product_name

-- 2NF 준수: product 정보를 별도 테이블로
CREATE TABLE products (
  id   BIGSERIAL PRIMARY KEY,
  name TEXT NOT NULL
);
```

### 제3정규형 (3NF)

2NF를 만족하면서, **이행 종속성**을 없앱니다. 기본키가 아닌 컬럼이 다른 비키 컬럼에 의존하면 분리합니다.

```sql
-- 위반 예시: zip_code가 city를 결정함 (이행 종속)
-- orders: order_id, customer_zip, customer_city

-- 3NF 준수: 주소 정보를 별도 테이블로
CREATE TABLE zip_codes (
  zip_code VARCHAR(10)  PRIMARY KEY,
  city     VARCHAR(100) NOT NULL,
  state    VARCHAR(50)  NOT NULL
);
```

> 실무에서는 3NF까지 정규화하는 것이 일반적이며, 읽기 성능이 중요한 경우 의도적으로 **비정규화(Denormalization)**를 적용하기도 합니다.

---

## 실전 설계 패턴

### 소프트 삭제 (Soft Delete)

데이터를 실제로 삭제하지 않고 `deleted_at` 컬럼으로 삭제 여부를 표시합니다. 복구 가능성 확보와 감사 추적에 유리합니다.

```sql
CREATE TABLE posts (
  id         BIGSERIAL    PRIMARY KEY,
  title      TEXT         NOT NULL,
  content    TEXT,
  created_at TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
  deleted_at TIMESTAMPTZ                           -- NULL이면 활성 상태
);

-- 삭제 처리
UPDATE posts SET deleted_at = NOW() WHERE id = 42;

-- 활성 포스트만 조회
SELECT * FROM posts WHERE deleted_at IS NULL;
```

### 감사 로그 (Audit Trail)

누가 언제 데이터를 변경했는지 추적하려면 별도의 이력 테이블을 사용합니다.

```sql
CREATE TABLE post_history (
  id         BIGSERIAL   PRIMARY KEY,
  post_id    BIGINT      NOT NULL REFERENCES posts(id),
  changed_by BIGINT      NOT NULL REFERENCES users(id),
  changed_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  old_data   JSONB,
  new_data   JSONB
);
```

### 계층 구조 (Adjacency List)

댓글의 대댓글처럼 계층 구조를 표현할 때 `parent_id`를 사용하는 인접 목록 패턴이 가장 단순합니다.

```sql
CREATE TABLE comments (
  id        BIGSERIAL PRIMARY KEY,
  post_id   BIGINT    REFERENCES posts(id),
  parent_id BIGINT    REFERENCES comments(id), -- 최상위 댓글은 NULL
  content   TEXT      NOT NULL,
  author_id BIGINT    NOT NULL REFERENCES users(id)
);
```

---

## 인덱스 전략

아무리 스키마가 잘 설계되어도 인덱스가 없으면 성능 문제를 피할 수 없습니다.

### 인덱스를 추가해야 할 때

- **WHERE 절에 자주 등장하는 컬럼**: 필터링에 사용되는 컬럼
- **JOIN 조건에 사용되는 외래키**: 외래키 컬럼에는 대부분 인덱스 필요
- **ORDER BY / GROUP BY에 사용되는 컬럼**: 정렬 성능 향상
- **고유성이 필요한 컬럼**: UNIQUE 인덱스로 무결성 보장

```sql
-- 복합 인덱스: 함께 조회되는 컬럼을 하나로
CREATE INDEX idx_orders_user_status
  ON orders (user_id, status);

-- 부분 인덱스: 특정 조건의 행만 인덱싱하여 크기 최소화
CREATE INDEX idx_orders_pending
  ON orders (created_at)
  WHERE status = 'pending';
```

### 과도한 인덱스 주의

인덱스는 읽기 성능을 올리지만 **쓰기 성능을 떨어뜨립니다**. INSERT/UPDATE/DELETE 시 모든 인덱스를 갱신해야 하기 때문입니다. 쓰기가 잦은 테이블에는 꼭 필요한 인덱스만 유지하세요.

---

## 설계 완료 전 체크리스트

- [ ] 모든 테이블에 `id` 기본키(BIGSERIAL 또는 UUID)가 있는가?
- [ ] `created_at`, `updated_at` 타임스탬프가 포함되어 있는가?
- [ ] 외래키 제약조건이 올바르게 설정되어 있는가?
- [ ] NOT NULL 제약조건이 필수 컬럼에 적용되어 있는가?
- [ ] N:M 관계가 중간 테이블로 해소되었는가?
- [ ] 자주 조회하는 컬럼에 적절한 인덱스가 있는가?
- [ ] 민감한 데이터(비밀번호, 카드번호 등)는 암호화 또는 해싱 처리되는가?

---

## 마치며

데이터베이스 설계는 한 번 잘못 굳으면 나중에 고치기가 매우 어렵습니다. 초기에 ERD를 그리고, 정규화를 적용하고, 인덱스 전략을 함께 고민하는 것이 장기적으로 훨씬 적은 비용을 만들어 냅니다.

좋은 스키마는 애플리케이션 코드를 단순하게 만들고, 쿼리를 직관적이게 하며, 데이터 무결성을 자동으로 지켜줍니다. 다음 프로젝트를 시작하기 전에 반드시 ERD를 먼저 그려보세요.
