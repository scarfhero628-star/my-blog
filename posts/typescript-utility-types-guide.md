---
title: TypeScript 유틸리티 타입 완전 가이드 — Partial, Pick, Omit, Record로 타입을 자유자재로 다루는 법
date: 2026-11-10
tags: [TypeScript, 유틸리티 타입, 타입 시스템, 웹 개발, 코드 품질]
description: TypeScript 내장 유틸리티 타입을 활용해 중복 없이 유연한 타입을 만들고 코드 품질을 높이는 실전 방법을 소개합니다.
---

TypeScript를 사용하다 보면 기존 타입을 조금씩 변형해야 하는 상황이 자주 생깁니다. API 요청 시 일부 필드만 선택적으로 보내거나, 특정 키만 뽑아내거나, 읽기 전용으로 만들어야 할 때처럼요. 이런 상황마다 새 타입을 처음부터 작성하면 중복 코드가 쌓이고 유지보수가 어려워집니다.

TypeScript는 이를 위해 **유틸리티 타입(Utility Types)** 을 기본으로 제공합니다. 유틸리티 타입은 기존 타입을 변환해 새로운 타입을 만드는 타입 레벨의 함수입니다. 한 번 익혀두면 타입 중복을 크게 줄이고 코드 의도를 명확하게 전달할 수 있습니다.

## 왜 유틸리티 타입이 필요한가

아래 예시를 보겠습니다.

```typescript
interface User {
  id: number;
  name: string;
  email: string;
  password: string;
  createdAt: Date;
}

// 사용자 수정 시: id, createdAt은 변경 불가 → 별도 타입 작성
interface UpdateUserDto {
  name?: string;
  email?: string;
  password?: string;
}

// API 응답에서 password 제외 → 또 별도 타입 작성
interface UserResponse {
  id: number;
  name: string;
  email: string;
  createdAt: Date;
}
```

`User` 하나를 두고 파생 타입 두 개를 손으로 작성했습니다. 필드가 추가되면 세 곳을 모두 고쳐야 합니다. 유틸리티 타입을 쓰면 `User` 하나만 관리하면 됩니다.

---

## 핵심 유틸리티 타입 7가지

### 1. `Partial<T>` — 모든 필드를 선택적으로

타입 `T`의 모든 프로퍼티를 옵셔널(`?`)로 만듭니다. 수정(PATCH) API나 폼 초기값에 유용합니다.

```typescript
type UpdateUserDto = Partial<Omit<User, 'id' | 'createdAt'>>;
// { name?: string; email?: string; password?: string; }

function updateUser(id: number, dto: Partial<User>) {
  // dto의 모든 필드가 선택적 → 일부만 보내도 OK
}
```

### 2. `Required<T>` — 모든 필드를 필수로

`Partial`의 반대입니다. 모든 옵셔널 프로퍼티를 필수로 바꿉니다. 서버에서 완성된 데이터를 반환할 때 타입을 강화하는 데 씁니다.

```typescript
interface Config {
  host?: string;
  port?: number;
  timeout?: number;
}

type StrictConfig = Required<Config>;
// { host: string; port: number; timeout: number; }
```

### 3. `Readonly<T>` — 읽기 전용 타입

모든 프로퍼티에 `readonly`를 붙입니다. 불변 객체를 표현할 때 적합합니다.

```typescript
const config: Readonly<Config> = { host: 'localhost', port: 3000 };
config.host = 'example.com'; // 오류: 읽기 전용 프로퍼티
```

> 참고: `Readonly`는 얕은(shallow) 불변입니다. 중첩 객체까지 완전히 막으려면 `DeepReadonly` 같은 커스텀 타입이 필요합니다.

### 4. `Pick<T, K>` — 원하는 키만 선택

타입 `T`에서 키 집합 `K`만 뽑아 새 타입을 만듭니다.

```typescript
type UserPreview = Pick<User, 'id' | 'name'>;
// { id: number; name: string; }

// 목록 API 응답처럼 일부 필드만 반환할 때
function getUserList(): UserPreview[] {
  return users.map(({ id, name }) => ({ id, name }));
}
```

### 5. `Omit<T, K>` — 특정 키를 제외

`Pick`의 반대로, 제외할 키를 명시합니다. 보안이 중요한 필드를 뺄 때 자주 씁니다.

```typescript
type UserResponse = Omit<User, 'password'>;
// { id: number; name: string; email: string; createdAt: Date; }

type CreateUserDto = Omit<User, 'id' | 'createdAt'>;
// { name: string; email: string; password: string; }
```

> `Omit`은 제외할 키가 적을 때, `Pick`은 선택할 키가 적을 때 더 간결합니다.

### 6. `Record<K, V>` — 키-값 맵 타입

키 타입 `K`와 값 타입 `V`로 객체 타입을 만듭니다. 딕셔너리나 룩업 테이블에 유용합니다.

```typescript
type Role = 'admin' | 'editor' | 'viewer';

const permissions: Record<Role, string[]> = {
  admin: ['read', 'write', 'delete'],
  editor: ['read', 'write'],
  viewer: ['read'],
};

// 키가 동적일 때도 활용
type UserMap = Record<number, User>;
const userMap: UserMap = { 1: alice, 2: bob };
```

### 7. `Exclude<T, U>` / `Extract<T, U>` — 유니온 타입 필터링

`Exclude`는 유니온 타입 `T`에서 `U`에 할당 가능한 타입을 제거합니다. `Extract`는 반대로 공통 타입만 추출합니다.

```typescript
type Status = 'pending' | 'active' | 'blocked' | 'deleted';

// 비활성 상태만 제거
type ActiveStatus = Exclude<Status, 'blocked' | 'deleted'>;
// 'pending' | 'active'

// 완료 상태만 추출
type TerminalStatus = Extract<Status, 'blocked' | 'deleted'>;
// 'blocked' | 'deleted'
```

---

## 실전 패턴 — 유틸리티 타입 조합하기

단독으로도 유용하지만, 조합할 때 진가가 드러납니다.

### CRUD API 타입 한 곳에서 관리하기

```typescript
interface Post {
  id: number;
  title: string;
  content: string;
  authorId: number;
  publishedAt: Date | null;
  createdAt: Date;
}

// 생성: id, 날짜 필드 제외
type CreatePostDto = Omit<Post, 'id' | 'createdAt' | 'publishedAt'>;

// 수정: 작성자와 날짜 필드 제외 후 선택적으로
type UpdatePostDto = Partial<Omit<Post, 'id' | 'authorId' | 'createdAt'>>;

// 목록 응답: content 제외 (미리보기)
type PostSummary = Omit<Post, 'content'>;
```

`Post` 인터페이스만 수정하면 파생 타입이 모두 자동으로 갱신됩니다.

### 함수 파라미터에서 타입 추출하기

`Parameters<T>`와 `ReturnType<T>`도 유용한 유틸리티 타입입니다.

```typescript
async function fetchUser(id: number, options: { cache: boolean }) {
  // ...
}

type FetchUserParams = Parameters<typeof fetchUser>;
// [id: number, options: { cache: boolean }]

type FetchUserResult = ReturnType<typeof fetchUser>;
// Promise<...>
```

---

## 자주 하는 실수

### `Omit`과 유니온 타입 주의

`Omit`은 유니온 타입에서 의도한 대로 동작하지 않을 수 있습니다.

```typescript
type A = { kind: 'a'; foo: string };
type B = { kind: 'b'; bar: string };

type AB = A | B;

// 의도: 각 멤버에서 'kind' 제거
// 실제: 공통 키만 연산에 포함됨
type WithoutKind = Omit<AB, 'kind'>;
// {} — 예상과 다름!

// 올바른 방법: 각 멤버에 개별 적용
type WithoutKindCorrect = Omit<A, 'kind'> | Omit<B, 'kind'>;
// { foo: string } | { bar: string }
```

### `Partial`은 중첩까지 처리하지 않는다

```typescript
interface Profile {
  user: { name: string; age: number };
  settings: { theme: string };
}

type PartialProfile = Partial<Profile>;
// { user?: { name: string; age: number }; settings?: { theme: string }; }
// user 내부는 여전히 필수!
```

중첩 객체까지 선택적으로 만들려면 재귀 유틸리티 타입을 직접 작성해야 합니다.

---

## 커스텀 유틸리티 타입 맛보기

내장 유틸리티 타입을 조합하면 프로젝트 전용 유틸리티 타입도 만들 수 있습니다.

```typescript
// 특정 키는 필수, 나머지는 선택적
type PickRequired<T, K extends keyof T> =
  Required<Pick<T, K>> & Partial<Omit<T, K>>;

type UserForm = PickRequired<User, 'name' | 'email'>;
// { name: string; email: string; id?: number; password?: string; ... }
```

이처럼 유틸리티 타입은 단순한 편의 기능을 넘어 타입 추상화의 도구입니다.

---

## 정리

| 유틸리티 타입 | 역할 |
|---|---|
| `Partial<T>` | 모든 필드를 선택적으로 |
| `Required<T>` | 모든 필드를 필수로 |
| `Readonly<T>` | 모든 필드를 읽기 전용으로 |
| `Pick<T, K>` | 특정 키만 선택 |
| `Omit<T, K>` | 특정 키를 제외 |
| `Record<K, V>` | 키-값 맵 타입 생성 |
| `Exclude<T, U>` | 유니온에서 타입 제거 |
| `Extract<T, U>` | 유니온에서 타입 추출 |
| `Parameters<T>` | 함수 파라미터 타입 추출 |
| `ReturnType<T>` | 함수 반환 타입 추출 |

TypeScript 유틸리티 타입을 적극 활용하면 타입 정의의 중복을 줄이고, 하나의 소스에서 파생 타입을 자동으로 유지할 수 있습니다. 처음에는 익숙하지 않더라도 `Partial`, `Pick`, `Omit` 세 가지만 먼저 실무에 적용해보세요. 타입 코드가 훨씬 간결해지는 것을 바로 느낄 수 있을 것입니다.
