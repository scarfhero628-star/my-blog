---
title: Zod로 타입 안전한 데이터 검증하기 — 런타임 유효성 검사의 새로운 표준
date: 2026-11-23
tags: [Zod, TypeScript, 스키마 검증, 런타임 검증, 웹 개발]
description: Zod 라이브러리로 TypeScript 타입과 런타임 유효성 검사를 동시에 처리해 더 안전하고 견고한 애플리케이션을 만드는 방법을 소개합니다.
---

## 왜 Zod인가?

TypeScript는 컴파일 타임에 타입 오류를 잡아주지만, **런타임에는 무력합니다**. 외부 API 응답, 사용자 입력, 환경 변수 — 이 모든 데이터는 TypeScript의 눈을 피해 애플리케이션 안으로 들어옵니다. 타입이 `string`이라고 선언해도 실제 값이 `null`이거나 엉뚱한 형태라면 런타임 에러는 피할 수 없습니다.

Zod는 이 문제를 우아하게 해결합니다. **스키마를 한 번만 정의하면** TypeScript 타입과 런타임 유효성 검사를 모두 얻을 수 있습니다.

```bash
npm install zod
```

---

## 기본 개념: 스키마와 타입 추론

Zod의 핵심은 `z` 객체로 스키마를 정의하고, `z.infer<>`로 TypeScript 타입을 자동 생성하는 것입니다.

```typescript
import { z } from 'zod';

const UserSchema = z.object({
  id: z.number(),
  name: z.string().min(1).max(50),
  email: z.string().email(),
  age: z.number().int().positive().optional(),
});

// 타입을 별도로 정의할 필요 없음
type User = z.infer<typeof UserSchema>;

// 런타임 검증
const result = UserSchema.safeParse({
  id: 1,
  name: '김철수',
  email: 'kim@example.com',
});

if (result.success) {
  console.log(result.data); // User 타입 보장
} else {
  console.error(result.error.issues); // 상세한 오류 목록
}
```

`parse()`는 검증 실패 시 예외를 던지고, `safeParse()`는 결과 객체를 반환합니다. 대부분의 경우 에러를 직접 처리할 수 있는 `safeParse()`가 더 안전합니다.

---

## 주요 스키마 타입

### 기본 타입

```typescript
z.string()        // 문자열
z.number()        // 숫자
z.boolean()       // 불리언
z.date()          // Date 객체
z.undefined()     // undefined
z.null()          // null
z.any()           // 모든 타입 (사용 최소화 권장)
z.unknown()       // 알 수 없는 타입 (any보다 안전)
```

### 문자열 유효성 검사

```typescript
const PasswordSchema = z
  .string()
  .min(8, '비밀번호는 8자 이상이어야 합니다')
  .max(100, '비밀번호는 100자 이하여야 합니다')
  .regex(/[A-Z]/, '대문자를 포함해야 합니다')
  .regex(/[0-9]/, '숫자를 포함해야 합니다');

const UrlSchema = z.string().url('올바른 URL 형식이 아닙니다');
const UuidSchema = z.string().uuid();
const EmailSchema = z.string().email();
```

### 숫자 유효성 검사

```typescript
const AgeSchema = z
  .number()
  .int('정수여야 합니다')
  .min(0, '0 이상이어야 합니다')
  .max(150, '150 이하여야 합니다');

const PriceSchema = z.number().positive().finite();
```

### 배열과 열거형

```typescript
// 배열
const TagsSchema = z.array(z.string()).min(1).max(10);

// 열거형
const RoleSchema = z.enum(['admin', 'editor', 'viewer']);
type Role = z.infer<typeof RoleSchema>; // 'admin' | 'editor' | 'viewer'

// 네이티브 열거형과 연동
enum Direction {
  Up = 'UP',
  Down = 'DOWN',
}
const DirectionSchema = z.nativeEnum(Direction);
```

---

## 고급 패턴

### 중첩 객체와 재사용

```typescript
const AddressSchema = z.object({
  street: z.string(),
  city: z.string(),
  zipCode: z.string().regex(/^\d{5}$/),
});

const CustomerSchema = z.object({
  id: z.number(),
  name: z.string(),
  address: AddressSchema,          // 중첩
  shippingAddress: AddressSchema.optional(), // 선택적 중첩
});
```

### 유니온과 판별 유니온

```typescript
// 단순 유니온
const IdSchema = z.union([z.string(), z.number()]);

// 판별 유니온 (discriminated union) — 더 효율적
const EventSchema = z.discriminatedUnion('type', [
  z.object({ type: z.literal('click'), x: z.number(), y: z.number() }),
  z.object({ type: z.literal('keydown'), key: z.string() }),
  z.object({ type: z.literal('scroll'), delta: z.number() }),
]);

type Event = z.infer<typeof EventSchema>;
```

### 변환 (Transform)

검증과 동시에 데이터를 변환할 수 있습니다.

```typescript
const DateStringSchema = z
  .string()
  .datetime()
  .transform((val) => new Date(val));

// string을 받아 Date를 반환
type ParsedDate = z.infer<typeof DateStringSchema>; // Date

const TrimmedStringSchema = z.string().trim().toLowerCase();

// API 응답의 snake_case를 camelCase로 변환
const ApiUserSchema = z
  .object({
    user_id: z.number(),
    full_name: z.string(),
    created_at: z.string().datetime(),
  })
  .transform((data) => ({
    userId: data.user_id,
    fullName: data.full_name,
    createdAt: new Date(data.created_at),
  }));
```

### 커스텀 검증 (refine)

```typescript
const PasswordConfirmSchema = z
  .object({
    password: z.string().min(8),
    confirmPassword: z.string(),
  })
  .refine((data) => data.password === data.confirmPassword, {
    message: '비밀번호가 일치하지 않습니다',
    path: ['confirmPassword'], // 오류 경로 지정
  });

// 비동기 검증
const UniqueEmailSchema = z.string().email().refine(
  async (email) => {
    const exists = await checkEmailExists(email);
    return !exists;
  },
  { message: '이미 사용 중인 이메일입니다' }
);
```

---

## 실전 활용 사례

### API 응답 검증

```typescript
const ApiResponseSchema = z.object({
  data: z.array(
    z.object({
      id: z.number(),
      title: z.string(),
      published: z.boolean(),
    })
  ),
  pagination: z.object({
    page: z.number(),
    total: z.number(),
  }),
});

async function fetchPosts() {
  const response = await fetch('/api/posts');
  const json = await response.json();

  const result = ApiResponseSchema.safeParse(json);
  if (!result.success) {
    // API 응답이 예상 형태가 아닐 때 명확한 오류
    throw new Error(`API 응답 형식 오류: ${result.error.message}`);
  }

  return result.data; // 타입 안전한 데이터
}
```

### 환경 변수 검증

서버 시작 시 환경 변수를 검증하면 잘못된 설정으로 인한 런타임 오류를 방지할 수 있습니다.

```typescript
const EnvSchema = z.object({
  NODE_ENV: z.enum(['development', 'production', 'test']),
  DATABASE_URL: z.string().url(),
  API_SECRET: z.string().min(32),
  PORT: z.coerce.number().int().min(1024).max(65535).default(3000),
});

// 앱 시작 시 한 번만 실행
const env = EnvSchema.parse(process.env);
export type Env = z.infer<typeof EnvSchema>;
export { env };
```

### React Hook Form과 통합

```typescript
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';

const SignupSchema = z.object({
  username: z.string().min(3).max(20),
  email: z.string().email(),
  password: z.string().min(8),
});

type SignupForm = z.infer<typeof SignupSchema>;

function SignupForm() {
  const { register, handleSubmit, formState: { errors } } = useForm<SignupForm>({
    resolver: zodResolver(SignupSchema),
  });

  return (
    <form onSubmit={handleSubmit((data) => console.log(data))}>
      <input {...register('username')} />
      {errors.username && <p>{errors.username.message}</p>}
      {/* ... */}
    </form>
  );
}
```

---

## Zod vs 대안 라이브러리

| 항목 | Zod | Yup | Joi |
|------|-----|-----|-----|
| TypeScript 지원 | 최우선 | 부분적 | 별도 타입 패키지 |
| 번들 크기 | ~13KB | ~18KB | ~25KB |
| 타입 추론 | 완벽 | 제한적 | 제한적 |
| 비동기 검증 | 지원 | 지원 | 지원 |
| 변환 기능 | 풍부 | 기본적 | 기본적 |

TypeScript 프로젝트라면 Zod가 압도적으로 유리합니다. 타입 정의와 검증 로직을 한 곳에서 관리할 수 있어 유지보수 비용이 크게 줄어듭니다.

---

## 마치며

Zod는 단순한 유효성 검사 라이브러리가 아닙니다. TypeScript 타입 시스템과 런타임 세계를 연결하는 다리 역할을 합니다. 스키마 하나로 타입 추론, 런타임 검증, 데이터 변환을 모두 처리할 수 있어 코드 중복을 줄이고 타입 안전성을 높일 수 있습니다.

특히 **외부 데이터가 들어오는 모든 경계** — API 응답, 사용자 입력, 환경 변수, 로컬 스토리지 — 에 Zod를 적용하면 "타입이 맞는데 왜 에러가 나지?"라는 디버깅 고통에서 해방될 수 있습니다.
