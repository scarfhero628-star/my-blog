---
title: MSW(Mock Service Worker) 완전 가이드 — API 모킹으로 프론트엔드 개발 생산성 높이기
date: 2026-10-29
tags: [MSW, API 모킹, 프론트엔드, 테스팅, 개발 생산성]
description: MSW를 활용해 실제 API 없이도 프론트엔드 개발과 테스트를 할 수 있는 API 모킹 전략을 소개합니다.
---

프론트엔드 개발을 하다 보면 백엔드 API가 아직 완성되지 않아 개발이 막히는 경험을 자주 합니다. 또는 테스트 코드를 작성할 때 실제 서버에 의존하면 테스트가 느려지고 불안정해지기 쉽습니다. **MSW(Mock Service Worker)** 는 이런 문제를 우아하게 해결해 주는 API 모킹 라이브러리입니다.

이 글에서는 MSW의 동작 원리부터 실전 설정, 개발 환경과 테스트 환경에서의 활용법까지 단계별로 정리합니다.

## MSW란 무엇인가?

MSW는 **Service Worker API** 를 활용해 브라우저 수준에서 네트워크 요청을 가로채 가짜 응답을 반환하는 라이브러리입니다. 기존의 API 모킹 방식과 달리 실제 HTTP 요청이 일어난 뒤 Service Worker가 응답을 대신 돌려주기 때문에, 애플리케이션 코드를 전혀 수정하지 않아도 됩니다.

### 기존 모킹 방식의 문제점

- **axios-mock-adapter** 같은 라이브러리는 `axios` 같은 특정 HTTP 클라이언트에 종속됩니다.
- `fetch`를 직접 패칭하면 코드가 지저분해지고 사이드 이펙트가 발생하기 쉽습니다.
- 실제 네트워크 레이어를 건드리지 않아 DevTools에서 요청을 확인할 수 없습니다.

MSW는 이 모든 문제를 해결합니다. 브라우저 DevTools의 Network 탭에서도 가짜 응답이 정상적으로 보이기 때문에 디버깅도 훨씬 편리합니다.

## 설치와 기본 설정

### 패키지 설치

```bash
npm install msw --save-dev
```

### 브라우저용 Service Worker 초기화

```bash
npx msw init public/ --save
```

이 명령어는 `public/mockServiceWorker.js` 파일을 생성합니다. 이 파일은 Service Worker의 진입점으로, 버전 관리에 포함시켜야 합니다.

### 핸들러 작성

`src/mocks/handlers.ts` 파일을 만들고 API 응답을 정의합니다.

```typescript
import { http, HttpResponse } from 'msw'

export const handlers = [
  // GET 요청 모킹
  http.get('/api/users', () => {
    return HttpResponse.json([
      { id: 1, name: '김철수', email: 'chulsoo@example.com' },
      { id: 2, name: '이영희', email: 'younghee@example.com' },
    ])
  }),

  // POST 요청 모킹
  http.post('/api/users', async ({ request }) => {
    const newUser = await request.json()
    return HttpResponse.json(
      { id: 3, ...newUser },
      { status: 201 }
    )
  }),

  // 에러 응답 시뮬레이션
  http.get('/api/products/:id', ({ params }) => {
    if (params.id === '999') {
      return HttpResponse.json(
        { message: '상품을 찾을 수 없습니다.' },
        { status: 404 }
      )
    }
    return HttpResponse.json({ id: params.id, name: '테스트 상품' })
  }),
]
```

## 브라우저 환경에서 사용하기

`src/mocks/browser.ts`를 생성합니다.

```typescript
import { setupWorker } from 'msw/browser'
import { handlers } from './handlers'

export const worker = setupWorker(...handlers)
```

애플리케이션 진입점(예: `main.tsx`)에서 개발 환경에서만 MSW를 활성화합니다.

```typescript
// main.tsx
async function enableMocking() {
  if (process.env.NODE_ENV !== 'development') return

  const { worker } = await import('./mocks/browser')
  return worker.start({
    onUnhandledRequest: 'warn', // 처리되지 않은 요청 경고
  })
}

enableMocking().then(() => {
  ReactDOM.createRoot(document.getElementById('root')!).render(
    <React.StrictMode>
      <App />
    </React.StrictMode>
  )
})
```

이제 `npm run dev`로 개발 서버를 시작하면 콘솔에 `[MSW] Mocking enabled.` 메시지가 출력되고, API 요청은 자동으로 MSW 핸들러가 처리합니다.

## 테스트 환경에서 사용하기

MSW는 Node.js 환경(Vitest, Jest 등)에서도 동작합니다.

### 서버 설정

`src/mocks/server.ts`를 생성합니다.

```typescript
import { setupServer } from 'msw/node'
import { handlers } from './handlers'

export const server = setupServer(...handlers)
```

### Vitest / Jest 설정

`setupTests.ts` 파일에서 서버 라이프사이클을 관리합니다.

```typescript
import { afterAll, afterEach, beforeAll } from 'vitest'
import { server } from './mocks/server'

// 테스트 시작 전 서버 구동
beforeAll(() => server.listen({ onUnhandledRequest: 'error' }))

// 각 테스트 후 핸들러 초기화 (테스트 간 격리)
afterEach(() => server.resetHandlers())

// 모든 테스트 후 서버 종료
afterAll(() => server.close())
```

### 테스트 작성 예시

```typescript
import { render, screen, waitFor } from '@testing-library/react'
import { server } from '../mocks/server'
import { http, HttpResponse } from 'msw'
import UserList from './UserList'

test('사용자 목록을 정상적으로 표시한다', async () => {
  render(<UserList />)

  await waitFor(() => {
    expect(screen.getByText('김철수')).toBeInTheDocument()
    expect(screen.getByText('이영희')).toBeInTheDocument()
  })
})

test('API 오류 시 에러 메시지를 표시한다', async () => {
  // 특정 테스트에서만 핸들러를 덮어씌움
  server.use(
    http.get('/api/users', () => {
      return HttpResponse.json(
        { message: '서버 오류' },
        { status: 500 }
      )
    })
  )

  render(<UserList />)

  await waitFor(() => {
    expect(screen.getByText('데이터를 불러올 수 없습니다.')).toBeInTheDocument()
  })
})
```

## 실전 활용 팁

### 1. 응답 지연 시뮬레이션

```typescript
import { delay, http, HttpResponse } from 'msw'

http.get('/api/slow-endpoint', async () => {
  await delay(2000) // 2초 지연
  return HttpResponse.json({ data: '느린 응답' })
})
```

로딩 스피너나 스켈레톤 UI가 제대로 동작하는지 확인할 때 유용합니다.

### 2. 네트워크 에러 시뮬레이션

```typescript
import { http, HttpResponse } from 'msw'

http.get('/api/flaky', () => {
  return HttpResponse.error() // 네트워크 자체 실패
})
```

### 3. 핸들러 모듈화

API가 많아지면 핸들러를 도메인별로 분리하면 관리하기 편합니다.

```
src/mocks/
  handlers/
    users.ts
    products.ts
    orders.ts
  index.ts   ← 모든 핸들러 통합
```

```typescript
// src/mocks/index.ts
import { userHandlers } from './handlers/users'
import { productHandlers } from './handlers/products'
import { orderHandlers } from './handlers/orders'

export const handlers = [
  ...userHandlers,
  ...productHandlers,
  ...orderHandlers,
]
```

## MSW를 도입해야 하는 이유

| 상황 | MSW 없이 | MSW 활용 |
|---|---|---|
| 백엔드 미완성 | 개발 블로킹 | 독립적으로 개발 가능 |
| 에러 케이스 테스트 | 서버 조작 필요 | 코드 한 줄로 시뮬레이션 |
| 테스트 속도 | 실서버 의존으로 느림 | 빠르고 안정적 |
| 디버깅 | 로그로만 확인 | DevTools Network 탭에서 확인 |

MSW는 한 번 설정해 두면 개발 환경과 테스트 환경 모두에서 **동일한 핸들러**를 재사용할 수 있어 유지보수 비용도 낮습니다. 백엔드 팀과 API 스펙을 먼저 합의한 뒤 MSW로 먼저 구현하면, 백엔드 완성 후 코드 한 줄 바꾸지 않고도 실제 API로 전환할 수 있습니다.

프론트엔드 테스트 커버리지를 높이고 싶거나, 백엔드 없이 빠르게 UI를 구현해야 한다면 MSW는 지금 당장 도입할 가치가 있는 도구입니다.
