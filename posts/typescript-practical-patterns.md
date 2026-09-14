---
title: TypeScript 실전 패턴 — 타입 안전성을 극대화하는 5가지 기법
date: 2026-09-14
tags: [TypeScript, 웹 개발, JavaScript, 코드 품질]
description: 실무에서 TypeScript를 더 잘 활용하기 위해 꼭 알아야 할 5가지 핵심 패턴과 실전 예제를 소개합니다.
---

## 왜 TypeScript 패턴을 배워야 할까?

TypeScript를 도입했지만 여전히 `any` 타입을 남발하거나, 런타임에서 예상치 못한 오류를 만나고 있다면 TypeScript를 제대로 활용하지 못하고 있을 가능성이 높습니다. 단순히 타입을 붙이는 것을 넘어, TypeScript의 고급 패턴을 익히면 컴파일 타임에 버그를 잡고 코드의 의도를 명확하게 표현할 수 있습니다.

이 글에서는 실무 코드를 더욱 견고하게 만드는 5가지 TypeScript 패턴을 소개합니다.

---

## 1. 유니온 타입과 타입 가드 — 런타임 타입 좁히기

여러 형태의 데이터를 다룰 때 유니온 타입과 타입 가드를 조합하면 안전하게 분기 처리를 할 수 있습니다.

```typescript
type SuccessResponse = { status: "success"; data: string };
type ErrorResponse = { status: "error"; message: string };
type ApiResponse = SuccessResponse | ErrorResponse;

function handleResponse(res: ApiResponse) {
  if (res.status === "success") {
    // 이 블록에서 res는 SuccessResponse로 좁혀짐
    console.log(res.data);
  } else {
    // 이 블록에서 res는 ErrorResponse로 좁혀짐
    console.error(res.message);
  }
}
```

`in` 연산자나 `instanceof`를 활용한 커스텀 타입 가드도 유용합니다:

```typescript
function isErrorResponse(res: ApiResponse): res is ErrorResponse {
  return res.status === "error";
}
```

`is` 키워드를 반환 타입에 사용하면 TypeScript가 분기 이후의 타입을 자동으로 추론해 줍니다.

---

## 2. 제네릭 — 재사용 가능한 타입 안전 코드

제네릭은 타입을 매개변수처럼 받아 다양한 타입에 동작하는 함수와 클래스를 만들 수 있게 합니다. `any`를 쓰는 대신 제네릭을 활용하면 타입 안전성을 유지하면서 유연성을 확보할 수 있습니다.

```typescript
// any 대신 제네릭 사용
function identity<T>(value: T): T {
  return value;
}

// 제약 조건(constraint) 추가
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

const user = { id: 1, name: "Alice" };
const name = getProperty(user, "name"); // 타입: string
```

`K extends keyof T`는 `K`가 반드시 `T`의 키 중 하나여야 함을 보장합니다. 잘못된 키를 전달하면 컴파일 에러가 발생합니다.

---

## 3. 맵드 타입 — 기존 타입에서 새 타입 파생

맵드 타입(Mapped Types)을 사용하면 기존 타입의 구조를 변환해 새로운 타입을 만들 수 있습니다. `Partial`, `Required`, `Readonly` 같은 유틸리티 타입이 이 패턴으로 만들어져 있습니다.

```typescript
// 모든 프로퍼티를 선택적으로 만드는 타입
type Optional<T> = {
  [K in keyof T]?: T[K];
};

// 특정 키만 선택적으로 만드는 타입
type PartialBy<T, K extends keyof T> = Omit<T, K> & Partial<Pick<T, K>>;

interface User {
  id: number;
  name: string;
  email: string;
}

// id는 필수, name과 email은 선택적
type UserUpdate = PartialBy<User, "name" | "email">;
```

이런 유틸리티 타입 조합은 API 요청/응답 타입 모델링, 폼 상태 관리, 부분 업데이트(PATCH) 처리에 특히 유용합니다.

---

## 4. Template Literal 타입 — 문자열 타입 정밀 제어

TypeScript 4.1부터 도입된 Template Literal 타입을 사용하면 문자열 패턴을 타입 수준에서 표현할 수 있습니다.

```typescript
type EventName = "click" | "focus" | "blur";
type EventHandler = `on${Capitalize<EventName>}`; // "onClick" | "onFocus" | "onBlur"

type CSSProperty = "margin" | "padding";
type CSSDirection = "Top" | "Right" | "Bottom" | "Left";
type CSSShorthand = `${CSSProperty}${CSSDirection}`;
// "marginTop" | "marginRight" | ... | "paddingLeft"

// 객체의 이벤트 핸들러 타입을 자동 생성
type EventMap<T extends string> = {
  [K in T as `on${Capitalize<K>}`]: () => void;
};
```

이 패턴은 이벤트 이름, CSS 클래스명, API 경로 등 일정한 규칙을 가진 문자열 집합을 타입으로 정의할 때 매우 강력합니다.

---

## 5. 조건부 타입 — 타입 수준의 if/else

조건부 타입(Conditional Types)은 `T extends U ? X : Y` 문법으로 입력 타입에 따라 다른 출력 타입을 반환합니다.

```typescript
type IsArray<T> = T extends any[] ? true : false;

type A = IsArray<string[]>; // true
type B = IsArray<string>;   // false

// 배열에서 요소 타입 추출
type ElementType<T> = T extends (infer U)[] ? U : never;
type NumArray = ElementType<number[]>; // number

// Promise에서 resolve 타입 추출
type Awaited<T> = T extends Promise<infer U> ? Awaited<U> : T;
type Result = Awaited<Promise<Promise<string>>>; // string
```

`infer` 키워드는 조건부 타입 안에서 타입을 추론하고 변수처럼 사용할 수 있게 해줍니다. 복잡한 제네릭 타입에서 내부 타입을 꺼낼 때 필수적입니다.

---

## 실무 적용 체크리스트

위 패턴들을 실무에 도입할 때 단계적으로 적용하면 부담이 줄어듭니다:

1. **`any` 제거 우선** — `unknown`으로 교체하고 타입 가드를 추가
2. **유틸리티 타입 활용** — `Partial`, `Required`, `Pick`, `Omit`으로 반복 타입 선언 줄이기
3. **제네릭으로 추상화** — 같은 구조의 함수/컴포넌트를 하나로 합치기
4. **Template Literal 타입** — 문자열 상수 집합을 타입으로 격상
5. **조건부 타입** — 복잡한 타입 변환이 필요할 때만 도입 (과도한 사용은 가독성을 떨어뜨림)

---

## 마치며

TypeScript의 진정한 가치는 단순히 타입을 붙이는 것이 아니라, 코드가 어떤 데이터를 기대하고 어떻게 변환하는지를 **타입 시스템으로 문서화**하는 데 있습니다. 처음에는 낯설게 느껴지더라도, 위 패턴들에 익숙해지면 런타임 오류가 줄어들고 리팩터링이 훨씬 안전해지는 것을 경험하게 됩니다.

한 번에 모든 패턴을 익히려 하기보다, 현재 프로젝트에서 가장 자주 겪는 타입 문제부터 하나씩 해결해 나가는 방식을 권장합니다.
