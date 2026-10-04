---
title: GraphQL 입문 가이드 — REST API 대신 GraphQL을 선택해야 할 이유
date: 2026-10-04
tags: [GraphQL, API, 웹 개발, 백엔드, 프론트엔드]
description: GraphQL의 핵심 개념과 REST API와의 차이를 실전 예제로 살펴보고, 언제 GraphQL을 도입해야 하는지 알아봅니다.
---

## GraphQL이란 무엇인가?

GraphQL은 2015년 Facebook(현 Meta)이 공개한 API 쿼리 언어이자 런타임입니다. REST API가 URL 기반으로 리소스를 노출하는 방식과 달리, GraphQL은 클라이언트가 필요한 데이터를 직접 명시하는 방식으로 동작합니다.

"필요한 데이터만 요청하고, 필요한 데이터만 받는다"는 철학이 GraphQL의 핵심입니다. 이 단순한 원칙 하나가 API 설계 방식을 근본적으로 바꾸어 놓습니다.

---

## REST API의 고질적인 문제들

GraphQL이 등장한 배경에는 REST API의 한계가 있습니다. 특히 모바일 앱과 다양한 화면 크기를 동시에 지원해야 하는 환경에서 두 가지 문제가 두드러집니다.

### Over-fetching — 너무 많이 받는 문제

REST API는 필요 이상의 데이터를 반환하는 경우가 많습니다.

```
// 사용자 이름만 필요하지만 모든 정보가 반환됩니다
GET /users/1
{
  "id": 1,
  "name": "홍길동",
  "email": "hong@example.com",
  "address": "서울시 강남구 ...",
  "createdAt": "2024-01-01T00:00:00Z",
  "updatedAt": "2026-09-01T12:00:00Z",
  "profileImageUrl": "https://...",
  "bio": "안녕하세요, 저는 ..."
}
```

모바일 앱에서 이름 하나를 표시하려고 수십 개 필드를 내려받는 것은 낭비입니다.

### Under-fetching — 여러 번 요청해야 하는 문제

반대로 한 번의 요청으로 필요한 데이터를 다 받지 못해 추가 요청을 해야 하는 경우도 있습니다.

```
// 사용자 프로필 화면에 사용자 정보 + 포스트 + 팔로워 수가 필요할 때
GET /users/1          → 사용자 기본 정보
GET /users/1/posts    → 포스트 목록
GET /users/1/followers → 팔로워 목록
```

세 번의 왕복이 발생하고, 네트워크 지연이 쌓여 화면이 늦게 나타납니다.

---

## GraphQL의 핵심 개념

### 스키마(Schema) — API의 계약서

GraphQL은 강타입 스키마로 API의 구조를 명확히 정의합니다. 이 스키마가 클라이언트와 서버 사이의 계약이 됩니다.

```graphql
type User {
  id: ID!
  name: String!
  email: String!
  posts: [Post]
  followers: [User]
}

type Post {
  id: ID!
  title: String!
  content: String!
  author: User!
  createdAt: String!
}

type Query {
  user(id: ID!): User
  posts(limit: Int): [Post]
}
```

`!`는 null이 될 수 없는 필드를 의미합니다. 스키마만 보면 어떤 데이터를 어떻게 요청할 수 있는지 한눈에 파악할 수 있습니다.

### 쿼리(Query) — 원하는 것만 요청하기

클라이언트는 필요한 필드만 골라서 요청합니다.

```graphql
# 사용자 이름과 최근 포스트 제목만 요청
query {
  user(id: "1") {
    name
    posts {
      title
      createdAt
    }
  }
}
```

응답 형태는 쿼리 형태를 그대로 따릅니다.

```json
{
  "data": {
    "user": {
      "name": "홍길동",
      "posts": [
        { "title": "GraphQL 입문", "createdAt": "2026-10-04" },
        { "title": "REST API 설계", "createdAt": "2026-09-24" }
      ]
    }
  }
}
```

단 한 번의 요청으로 사용자 정보와 포스트 목록을 동시에 가져왔습니다.

### 뮤테이션(Mutation) — 데이터 변경하기

데이터를 생성·수정·삭제할 때는 `mutation`을 사용합니다.

```graphql
mutation CreatePost($input: CreatePostInput!) {
  createPost(input: $input) {
    id
    title
    createdAt
  }
}
```

```json
// 변수
{
  "input": {
    "title": "GraphQL 입문",
    "content": "오늘은 GraphQL을 배워봅시다.",
    "authorId": "1"
  }
}
```

변수를 분리하면 쿼리를 재사용하기 쉽고, 인젝션 공격도 방지할 수 있습니다.

### 구독(Subscription) — 실시간 데이터

채팅, 알림 같은 실시간 기능이 필요할 때는 `subscription`을 사용합니다.

```graphql
subscription {
  messageAdded(roomId: "general") {
    id
    content
    author {
      name
    }
    createdAt
  }
}
```

WebSocket을 통해 서버의 변경 사항이 생기면 즉시 클라이언트에 푸시됩니다.

---

## JavaScript에서 GraphQL 시작하기

### 서버 구현 — Apollo Server

```javascript
import { ApolloServer } from '@apollo/server';
import { startStandaloneServer } from '@apollo/server/standalone';

const typeDefs = `
  type User {
    id: ID!
    name: String!
    email: String!
  }

  type Query {
    users: [User!]!
    user(id: ID!): User
  }
`;

const users = [
  { id: '1', name: '홍길동', email: 'hong@example.com' },
  { id: '2', name: '김영희', email: 'kim@example.com' },
];

const resolvers = {
  Query: {
    users: () => users,
    user: (_, { id }) => users.find(u => u.id === id) ?? null,
  },
};

const server = new ApolloServer({ typeDefs, resolvers });
const { url } = await startStandaloneServer(server, { listen: { port: 4000 } });
console.log(`서버 실행 중: ${url}`);
```

`resolvers`는 스키마의 각 필드를 실제 데이터와 연결하는 함수 모음입니다.

### 클라이언트 구현 — Apollo Client + React

```javascript
import { useQuery, gql } from '@apollo/client';

const GET_USER = gql`
  query GetUser($id: ID!) {
    user(id: $id) {
      name
      email
    }
  }
`;

function UserProfile({ userId }) {
  const { loading, error, data } = useQuery(GET_USER, {
    variables: { id: userId },
  });

  if (loading) return <p>로딩 중...</p>;
  if (error) return <p>오류: {error.message}</p>;

  return (
    <div>
      <h2>{data.user.name}</h2>
      <p>{data.user.email}</p>
    </div>
  );
}
```

Apollo Client는 요청 결과를 자동으로 캐시하므로 같은 사용자 정보를 여러 컴포넌트에서 써도 네트워크 요청이 한 번만 일어납니다.

---

## GraphQL vs REST — 언제 무엇을 선택할까?

| 상황 | 추천 |
|------|------|
| 다양한 클라이언트(웹·iOS·Android)가 서로 다른 데이터를 필요로 할 때 | GraphQL |
| 단순한 CRUD API | REST |
| 복잡하게 연결된 데이터를 한 번에 조회해야 할 때 | GraphQL |
| 공개 API로 HTTP 캐싱이 중요할 때 | REST |
| 실시간 기능(채팅·알림)이 핵심일 때 | GraphQL Subscription |
| 팀이 GraphQL에 익숙하지 않고 빠른 출시가 목표일 때 | REST |

---

## 도입 전에 알아야 할 주의사항

### N+1 문제

포스트 목록을 가져올 때 각 포스트의 작성자를 개별 DB 쿼리로 조회하면, 포스트 100개에 DB 쿼리가 101번 발생합니다. **DataLoader** 라이브러리로 배치 처리해 해결하세요.

```javascript
import DataLoader from 'dataloader';

const userLoader = new DataLoader(async (ids) => {
  const users = await db.users.findByIds(ids);
  return ids.map(id => users.find(u => u.id === id));
});
```

### 쿼리 복잡도 제한

클라이언트가 무한히 중첩된 쿼리(`users → posts → author → posts → author → ...`)를 보내 서버에 부하를 줄 수 있습니다. `graphql-depth-limit`으로 중첩 깊이를 제한하세요.

```javascript
import depthLimit from 'graphql-depth-limit';

const server = new ApolloServer({
  typeDefs,
  resolvers,
  validationRules: [depthLimit(5)], // 최대 5단계까지만 허용
});
```

### 캐싱 전략

REST는 URL마다 HTTP 캐시가 자연스럽게 동작하지만, GraphQL은 단일 엔드포인트에 POST 요청을 사용해 브라우저 캐시가 적용되지 않습니다. Apollo Client의 `InMemoryCache`와 Persisted Queries를 활용해 클라이언트 캐싱을 구성하세요.

---

## 마치며

GraphQL은 모든 상황에 맞는 만능 해결책이 아닙니다. 그러나 다양한 클라이언트가 각기 다른 데이터를 요구하는 복잡한 애플리케이션에서는 REST보다 훨씬 유연하고 효율적입니다.

처음 API를 설계할 때는 REST로 시작해 클라이언트의 데이터 요구사항이 복잡해지는 시점에 GraphQL 도입을 검토하는 것도 현실적인 전략입니다. 중요한 것은 도구가 아니라 문제에 맞는 선택입니다.

**더 공부하고 싶다면:**

- [GraphQL 공식 문서](https://graphql.org/learn/)
- [Apollo GraphQL 공식 문서](https://www.apollographql.com/docs/)
- [How to GraphQL](https://www.howtographql.com/) — 풀스택 GraphQL 튜토리얼
