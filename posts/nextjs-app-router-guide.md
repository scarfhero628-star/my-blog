---
title: Next.js App Router 완전 가이드 — Pages Router에서 App Router로 넘어가야 하는 이유
date: 2026-10-05
tags: [Next.js, React, 웹 개발, SSR, 프론트엔드]
description: Next.js 13부터 도입된 App Router의 핵심 개념과 Pages Router와의 차이점을 실전 예제로 살펴봅니다.
---

Next.js 13에서 공개된 App Router는 단순한 기능 추가가 아니라, 프레임워크의 철학 자체를 바꾼 패러다임 전환입니다. React Server Components를 중심에 두고 레이아웃·데이터 페칭·라우팅 방식을 전면 재설계했습니다. 기존 Pages Router 프로젝트를 운영하고 있거나 Next.js를 막 시작하려는 개발자라면, App Router가 왜 더 나은 선택인지 이해해두는 것이 중요합니다.

## App Router가 필요한 배경

Pages Router 시절에는 데이터 페칭을 `getServerSideProps`, `getStaticProps`, `getStaticPaths` 세 가지 함수로 처리했습니다. 이 방식은 페이지 단위로만 동작했기 때문에 컴포넌트 깊숙한 곳에서 서버 데이터가 필요할 때마다 props를 통해 내려보내야 했고, 코드가 복잡해졌습니다.

App Router는 이 문제를 **React Server Components(RSC)** 로 해결합니다. 이제 어떤 컴포넌트든 async 함수로 만들어 서버에서 직접 데이터를 가져올 수 있습니다.

## App Router의 디렉터리 구조

App Router는 `app/` 디렉터리를 사용합니다. 파일 이름이 라우팅 규칙을 결정합니다.

```
app/
  layout.tsx        # 루트 레이아웃 (필수)
  page.tsx          # 홈페이지 (/)
  about/
    page.tsx        # /about
  blog/
    [slug]/
      page.tsx      # /blog/:slug
  dashboard/
    layout.tsx      # 대시보드 전용 레이아웃
    page.tsx        # /dashboard
    settings/
      page.tsx      # /dashboard/settings
```

### 예약 파일 이름

| 파일명 | 역할 |
|---|---|
| `page.tsx` | 해당 경로의 UI |
| `layout.tsx` | 공유 레이아웃, 리렌더링 없이 유지됨 |
| `loading.tsx` | Suspense 기반 로딩 UI |
| `error.tsx` | 에러 바운더리 |
| `not-found.tsx` | 404 UI |

## 서버 컴포넌트와 클라이언트 컴포넌트

App Router에서 모든 컴포넌트는 기본적으로 **서버 컴포넌트**입니다. `"use client"` 지시문을 파일 맨 위에 추가해야만 클라이언트 컴포넌트가 됩니다.

### 서버 컴포넌트 — 데이터 페칭

```tsx
// app/blog/[slug]/page.tsx
// async 함수 자체가 서버에서 실행됩니다.
async function BlogPost({ params }: { params: { slug: string } }) {
  const post = await fetch(`https://api.example.com/posts/${params.slug}`, {
    next: { revalidate: 60 }, // 60초 간격으로 ISR
  }).then((res) => res.json());

  return (
    <article>
      <h1>{post.title}</h1>
      <p>{post.content}</p>
    </article>
  );
}

export default BlogPost;
```

별도의 `getServerSideProps`가 필요 없습니다. 컴포넌트 자체가 서버에서 렌더링되며, fetch 결과는 자동으로 캐싱됩니다.

### 클라이언트 컴포넌트 — 인터랙션

```tsx
"use client";

import { useState } from "react";

export function LikeButton({ initialCount }: { initialCount: number }) {
  const [count, setCount] = useState(initialCount);

  return (
    <button onClick={() => setCount((c) => c + 1)}>
      ❤️ {count}
    </button>
  );
}
```

`useState`, `useEffect`, 이벤트 핸들러처럼 브라우저 환경이 필요한 코드는 클라이언트 컴포넌트에만 작성합니다.

## 레이아웃 중첩

App Router의 강력한 기능 중 하나는 **중첩 레이아웃**입니다. 각 경로 세그먼트마다 레이아웃을 정의할 수 있고, 자식 경로로 이동해도 부모 레이아웃은 리렌더링되지 않습니다.

```tsx
// app/dashboard/layout.tsx
export default function DashboardLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <div className="flex">
      <Sidebar />
      <main className="flex-1">{children}</main>
    </div>
  );
}
```

`/dashboard`와 `/dashboard/settings` 사이를 이동해도 `Sidebar` 컴포넌트는 다시 마운트되지 않습니다. 사이드바의 스크롤 위치나 상태가 유지된다는 의미입니다.

## 데이터 페칭 전략

### 캐싱 옵션

Next.js의 `fetch`는 세 가지 캐싱 전략을 지원합니다.

```tsx
// 정적 생성 (빌드 시 한 번만 실행)
fetch(url, { cache: "force-cache" });

// 서버 사이드 렌더링 (매 요청마다 새로 실행)
fetch(url, { cache: "no-store" });

// ISR (지정한 초마다 백그라운드 재생성)
fetch(url, { next: { revalidate: 3600 } });
```

### 병렬 데이터 페칭

여러 데이터를 동시에 가져올 때는 `Promise.all`을 활용합니다.

```tsx
async function Page() {
  const [user, posts] = await Promise.all([
    fetchUser(),
    fetchPosts(),
  ]);

  return <Profile user={user} posts={posts} />;
}
```

순차적으로 페칭하면 waterfall 문제가 생기므로 가능한 병렬로 처리하는 것이 성능에 유리합니다.

## Pages Router에서 마이그레이션하기

App Router로의 전환은 점진적으로 진행할 수 있습니다. `app/` 디렉터리와 `pages/` 디렉터리는 공존이 가능합니다.

1. `app/layout.tsx`를 먼저 만들어 공통 레이아웃을 이전합니다.
2. 새로운 페이지는 `app/` 디렉터리에 작성합니다.
3. 기존 `pages/` 페이지는 여유 있게 마이그레이션합니다.
4. `getServerSideProps`는 서버 컴포넌트의 직접 fetch로, `getStaticProps`는 `cache: "force-cache"` fetch로 대체합니다.

## 마무리

App Router는 처음에는 낯설지만, 익숙해지면 코드가 훨씬 간결해집니다. 서버에서 처리할 것과 클라이언트에서 처리할 것을 명확히 구분하는 사고방식이 핵심입니다. 서버 컴포넌트로 최대한 많은 작업을 서버에서 처리하고, 실제로 인터랙션이 필요한 부분만 클라이언트 컴포넌트로 작성하는 패턴을 습관화해보세요. 번들 크기가 줄어들고 초기 로딩 성능이 눈에 띄게 개선될 것입니다.
