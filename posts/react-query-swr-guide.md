---
title: React Query와 SWR로 서버 상태 관리하기 — 데이터 페칭의 새로운 표준
date: 2026-10-23
tags: [React Query, SWR, 서버 상태, React, 데이터 페칭]
description: React Query와 SWR을 활용해 서버 상태를 효율적으로 관리하고 캐싱·재검증·낙관적 업데이트를 구현하는 실전 방법을 소개합니다.
---

서버에서 데이터를 불러오는 일은 프론트엔드 개발의 핵심 과제 중 하나입니다. 로딩 상태 표시, 오류 처리, 캐싱, 재시도, 폴링… 이 모든 걸 `useEffect`와 `useState`로 직접 구현하다 보면 코드는 금세 복잡해집니다. **React Query**와 **SWR**은 이 문제를 우아하게 해결해 주는 서버 상태 관리 라이브러리입니다.

## 클라이언트 상태 vs. 서버 상태

상태 관리 라이브러리를 고를 때 가장 먼저 구분해야 하는 것이 있습니다.

- **클라이언트 상태**: UI 토글, 모달 열림/닫힘, 폼 입력값 등 브라우저에서만 존재하는 상태
- **서버 상태**: 백엔드 API에서 가져오는 데이터 — 항상 비동기이고, 외부에서 변경될 수 있으며, 신선도(staleness)를 고려해야 합니다

Redux나 Zustand는 클라이언트 상태에 강합니다. 반면 React Query와 SWR은 서버 상태에 특화돼 있어, **두 라이브러리를 함께 사용하는 것**이 현대적인 접근 방식입니다.

---

## SWR 기초

SWR은 Vercel이 만든 경량 데이터 페칭 라이브러리입니다. 이름 자체가 HTTP 캐시 전략인 `stale-while-revalidate`의 약자로, "오래된 데이터를 먼저 보여주고 백그라운드에서 최신 데이터로 갱신한다"는 원칙을 따릅니다.

### 기본 사용법

```bash
npm install swr
```

```tsx
import useSWR from 'swr';

const fetcher = (url: string) => fetch(url).then(res => res.json());

function UserProfile({ userId }: { userId: string }) {
  const { data, error, isLoading } = useSWR(`/api/users/${userId}`, fetcher);

  if (isLoading) return <p>로딩 중...</p>;
  if (error) return <p>오류가 발생했습니다.</p>;

  return <h1>{data.name}</h1>;
}
```

단 몇 줄로 로딩 상태, 오류 처리, 캐싱이 모두 처리됩니다. 같은 키(`/api/users/1`)를 사용하는 컴포넌트가 여러 개 있어도 네트워크 요청은 한 번만 발생합니다.

### SWR의 자동 재검증

SWR은 다음 시점에 자동으로 데이터를 다시 불러옵니다.

- 브라우저 탭이 다시 포커스될 때
- 네트워크가 재연결될 때
- `refreshInterval` 옵션으로 지정한 주기마다 (폴링)

```tsx
// 5초마다 데이터를 갱신하는 실시간 대시보드
const { data } = useSWR('/api/metrics', fetcher, {
  refreshInterval: 5000,
});
```

---

## React Query 심화

React Query(현재 TanStack Query)는 SWR보다 더 풍부한 기능을 제공합니다. 복잡한 캐시 관리, 낙관적 업데이트, 무한 스크롤, 서버 사이드 렌더링 지원 등이 필요할 때 탁월한 선택입니다.

### 설정과 기본 쿼리

```bash
npm install @tanstack/react-query
```

```tsx
// main.tsx
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 1000 * 60 * 5, // 5분 동안 fresh 상태 유지
      retry: 2,
    },
  },
});

function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <MyApp />
    </QueryClientProvider>
  );
}
```

```tsx
// 데이터 읽기
import { useQuery } from '@tanstack/react-query';

function PostList() {
  const { data, isPending, isError } = useQuery({
    queryKey: ['posts'],
    queryFn: () => fetch('/api/posts').then(res => res.json()),
  });

  if (isPending) return <Spinner />;
  if (isError) return <ErrorMessage />;

  return data.map(post => <PostCard key={post.id} post={post} />);
}
```

### Mutation과 낙관적 업데이트

서버 데이터를 변경할 때는 `useMutation`을 사용합니다. **낙관적 업데이트(Optimistic Update)**를 적용하면 서버 응답을 기다리지 않고 UI를 즉시 업데이트해 사용자 경험을 크게 향상시킬 수 있습니다.

```tsx
import { useMutation, useQueryClient } from '@tanstack/react-query';

function LikeButton({ postId }: { postId: number }) {
  const queryClient = useQueryClient();

  const mutation = useMutation({
    mutationFn: () => fetch(`/api/posts/${postId}/like`, { method: 'POST' }),
    onMutate: async () => {
      // 진행 중인 쿼리를 취소해 충돌 방지
      await queryClient.cancelQueries({ queryKey: ['posts', postId] });

      // 현재 캐시값 백업
      const previous = queryClient.getQueryData(['posts', postId]);

      // 낙관적으로 캐시 업데이트
      queryClient.setQueryData(['posts', postId], (old: Post) => ({
        ...old,
        likes: old.likes + 1,
      }));

      return { previous };
    },
    onError: (err, _, context) => {
      // 실패 시 이전 값으로 롤백
      queryClient.setQueryData(['posts', postId], context?.previous);
    },
    onSettled: () => {
      // 성공/실패 모두 최신 데이터로 동기화
      queryClient.invalidateQueries({ queryKey: ['posts', postId] });
    },
  });

  return <button onClick={() => mutation.mutate()}>좋아요</button>;
}
```

---

## React Query vs. SWR — 언제 무엇을 쓸까?

| 기준 | SWR | React Query |
|------|-----|------------|
| 번들 크기 | ~4KB | ~12KB |
| 러닝 커브 | 낮음 | 중간 |
| 낙관적 업데이트 | 수동 구현 필요 | 내장 지원 |
| 무한 스크롤 | 기본 지원 | 더 세밀한 제어 |
| 개발자 도구 | 없음 | React Query DevTools |
| 캐시 무효화 | 간단 | 정교한 전략 가능 |

**SWR**은 단순한 데이터 페칭 위주의 프로젝트, 빠른 프로토타이핑, 번들 크기가 중요한 경우에 적합합니다. **React Query**는 복잡한 변이(mutation) 로직, 낙관적 업데이트, 정교한 캐시 제어가 필요한 대규모 애플리케이션에 어울립니다.

---

## 실전 팁

### 쿼리 키를 배열로 구조화하기

```tsx
// 나쁜 예: 문자열 키
useQuery({ queryKey: 'user-posts-1' });

// 좋은 예: 계층적 배열 키
useQuery({ queryKey: ['users', userId, 'posts', { page }] });
```

배열 키를 쓰면 `invalidateQueries(['users', userId])`로 특정 사용자의 모든 관련 쿼리를 한 번에 무효화할 수 있습니다.

### staleTime과 cacheTime 이해하기

- **staleTime**: 데이터가 '신선'한 상태로 유지되는 시간. 이 안에는 재검증하지 않습니다.
- **gcTime(구 cacheTime)**: 쿼리가 사용되지 않을 때 캐시에 보관되는 시간.

자주 바뀌지 않는 설정값은 `staleTime: Infinity`로, 실시간 데이터는 `staleTime: 0`으로 설정합니다.

---

서버 상태 관리를 직접 구현하던 시절과 비교하면, React Query나 SWR은 수백 줄의 보일러플레이트를 단 몇 줄로 압축해 줍니다. 다음 프로젝트에서 `useEffect`로 데이터를 불러오려 한다면, 먼저 이 두 라이브러리를 검토해 보세요.
