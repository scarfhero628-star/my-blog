---
title: Vite로 개발 환경을 혁신하다 — 번개처럼 빠른 모던 빌드 도구 입문
date: 2026-10-06
tags: [Vite, 빌드 도구, 웹 개발, 프론트엔드, 번들러]
description: 기존 번들러의 느린 빌드 문제를 해결한 Vite의 핵심 원리와 실전 설정법을 소개합니다.
---

## 왜 Vite인가?

프론트엔드 개발을 하다 보면 누구나 한 번쯤 이런 경험을 한다. 파일 하나를 수정했는데 브라우저에 반영되기까지 몇 초씩 기다려야 하는 답답함. 프로젝트가 커질수록 더 심해지는 이 문제는 Webpack 같은 전통적인 번들러가 가진 구조적 한계에서 비롯된다.

**Vite**(프랑스어로 '빠르다'는 뜻)는 이 문제를 근본적으로 다르게 접근한다. 2020년 Vue.js 창시자 에반 유(Evan You)가 발표한 Vite는 현재 프론트엔드 생태계에서 가장 빠르게 성장하는 빌드 도구 중 하나다.

---

## Vite가 빠른 이유

### 기존 번들러의 문제점

Webpack, Parcel 같은 기존 번들러는 **번들 기반(Bundle-based)** 방식으로 동작한다. 개발 서버를 시작하면:

1. 전체 애플리케이션 코드를 분석
2. 의존성 그래프를 생성
3. 모든 모듈을 하나(또는 여러 개)의 번들로 묶음
4. 그 번들을 브라우저에 제공

프로젝트 규모가 작을 때는 괜찮지만, 모듈이 수백 개, 수천 개로 늘어나면 초기 서버 시작 시간이 수십 초를 넘기기도 한다.

### Vite의 접근 방식: ESM + 사전 번들링

Vite는 두 가지 핵심 기술로 이 문제를 해결한다.

**① 네이티브 ES Modules(ESM) 활용**

현대 브라우저는 `import/export` 문법을 직접 이해한다. Vite는 개발 환경에서 번들링을 하지 않고, 브라우저가 필요한 모듈을 직접 요청하게 한다.

```javascript
// 브라우저가 이 import를 직접 처리
import { createApp } from '/node_modules/.vite/deps/vue.js'
import App from '/src/App.vue'

createApp(App).mount('#app')
```

서버는 요청이 들어올 때만 해당 파일을 변환(transform)하면 되므로 초기 시작 시간이 극적으로 줄어든다.

**② esbuild를 이용한 사전 번들링**

`node_modules` 안의 패키지들은 여전히 CommonJS 형식이거나 수백 개의 파일로 쪼개진 경우가 많다. Vite는 이 의존성들을 **esbuild**(Go로 작성된 초고속 번들러)로 미리 한 번만 번들링해 캐시한다. esbuild는 Webpack보다 10~100배 빠르다.

---

## Vite 시작하기

### 프로젝트 생성

```bash
# npm
npm create vite@latest my-app

# yarn
yarn create vite my-app

# pnpm
pnpm create vite my-app
```

템플릿 선택 화면에서 원하는 프레임워크와 언어를 고른다.

```
✔ Select a framework: › React
✔ Select a variant: › TypeScript
```

```bash
cd my-app
npm install
npm run dev
```

개발 서버가 뜨는 데 1초도 걸리지 않는 것을 확인할 수 있다.

### 디렉토리 구조

```
my-app/
├── public/          # 정적 파일 (빌드 시 그대로 복사)
├── src/
│   ├── assets/
│   ├── App.tsx
│   └── main.tsx
├── index.html       # 엔트리 포인트 (public이 아닌 루트에 위치!)
├── vite.config.ts
└── package.json
```

Webpack과 달리 `index.html`이 프로젝트 루트에 있다. Vite가 HTML 파일을 진입점으로 직접 읽기 때문이다.

---

## vite.config.ts 핵심 설정

```typescript
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import path from 'path'

export default defineConfig({
  plugins: [react()],

  // 경로 별칭 설정
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
    },
  },

  // 개발 서버 설정
  server: {
    port: 3000,
    open: true,       // 브라우저 자동 열기
    proxy: {
      '/api': {
        target: 'http://localhost:8080',
        changeOrigin: true,
      },
    },
  },

  // 빌드 설정
  build: {
    outDir: 'dist',
    sourcemap: true,
    rollupOptions: {
      output: {
        // 청크 분리 전략
        manualChunks: {
          vendor: ['react', 'react-dom'],
        },
      },
    },
  },
})
```

### 경로 별칭(@) 사용하기

```typescript
// Before: 깊은 상대 경로
import Button from '../../../components/common/Button'

// After: 명확한 절대 경로
import Button from '@/components/common/Button'
```

---

## Hot Module Replacement(HMR)

HMR은 전체 페이지를 새로고침하지 않고, 변경된 모듈만 실시간으로 교체하는 기술이다.

Vite의 HMR은 번들러 없이 ESM 단위로 작동하기 때문에 **변경된 파일만** 브라우저로 전송한다. 파일 저장 후 브라우저 반영까지 걸리는 시간이 수십 ms 수준이다.

React나 Vue 플러그인을 사용하면 컴포넌트 상태를 유지한 채로 UI를 업데이트하는 **Fast Refresh**도 기본 지원된다.

---

## 환경 변수 관리

Vite는 `.env` 파일로 환경 변수를 관리하며, 클라이언트에 노출할 변수는 반드시 `VITE_` 접두사를 붙여야 한다.

```
# .env
VITE_API_URL=https://api.example.com
VITE_APP_NAME=My Awesome App
DB_PASSWORD=secret   # 이건 클라이언트에 노출되지 않음
```

```typescript
// 코드에서 사용
const apiUrl = import.meta.env.VITE_API_URL
const appName = import.meta.env.VITE_APP_NAME
```

---

## 프로덕션 빌드

Vite는 프로덕션 빌드에는 **Rollup**을 사용한다. Rollup은 트리 쉐이킹과 코드 분할에 최적화되어 있어 최종 번들 크기를 최소화한다.

```bash
npm run build
```

빌드 결과물을 로컬에서 미리 볼 수 있다.

```bash
npm run preview
```

### 빌드 결과 분석

번들 크기가 궁금하다면 `rollup-plugin-visualizer`를 사용해보자.

```bash
npm install -D rollup-plugin-visualizer
```

```typescript
// vite.config.ts
import { visualizer } from 'rollup-plugin-visualizer'

export default defineConfig({
  plugins: [
    react(),
    visualizer({ open: true }), // 빌드 후 자동으로 분석 페이지 열기
  ],
})
```

---

## Vite vs Webpack: 언제 무엇을 쓸까?

| 기준 | Vite | Webpack |
|------|------|---------|
| 개발 서버 시작 | 매우 빠름 (< 1초) | 느림 (수~수십 초) |
| HMR 속도 | 매우 빠름 | 보통 |
| 생태계 성숙도 | 빠르게 성장 중 | 매우 성숙 |
| 설정 복잡도 | 낮음 | 높음 |
| 레거시 브라우저 지원 | 플러그인 필요 | 강력 지원 |
| 대규모 엔터프라이즈 | 검증 중 | 검증됨 |

신규 프로젝트라면 Vite를, 레거시 브라우저 지원이 필수이거나 이미 Webpack 설정이 잘 갖춰진 프로젝트라면 굳이 마이그레이션하지 않아도 된다.

---

## 마치며

Vite는 단순히 빠른 빌드 도구가 아니다. **개발자 경험(DX)을 근본적으로 개선**하는 도구다. 파일을 저장하고 브라우저에 반영되기까지의 그 짧은 시간이 하루에 수백 번 쌓이면 생산성의 차이가 된다.

새 프로젝트를 시작한다면 Vite를 기본 선택지로 고려해보자. 공식 문서(`vitejs.dev`)도 훌륭하게 잘 정리되어 있다.
