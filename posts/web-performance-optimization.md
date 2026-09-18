---
title: 웹 성능 최적화 완전 가이드 — Core Web Vitals부터 번들 최적화까지
date: 2026-09-18
tags: [웹 성능, Core Web Vitals, 최적화, 웹 개발]
description: Core Web Vitals 개선을 중심으로 실무에서 바로 적용할 수 있는 웹 성능 최적화 기법을 총정리합니다.
---

느린 웹사이트는 사용자 이탈과 직결됩니다. Google 연구에 따르면 페이지 로딩 시간이 1초에서 3초로 늘어나면 이탈률이 32% 증가합니다. 더불어 Core Web Vitals는 Google 검색 순위에도 직접 영향을 미칩니다. 이 글에서는 실무에서 즉시 적용할 수 있는 웹 성능 최적화 기법을 체계적으로 정리합니다.

## Core Web Vitals 이해하기

Google이 제시하는 세 가지 핵심 지표를 먼저 파악해야 합니다.

### LCP (Largest Contentful Paint)

페이지에서 가장 큰 콘텐츠 요소가 화면에 렌더링되는 시간입니다. 주로 히어로 이미지, 큰 텍스트 블록이 대상입니다.

- **좋음**: 2.5초 이하
- **개선 필요**: 2.5~4.0초
- **나쁨**: 4.0초 초과

LCP를 개선하려면 히어로 이미지에 `fetchpriority="high"` 속성을 추가하고, 서버 응답 시간(TTFB)을 줄이는 것이 가장 효과적입니다.

```html
<!-- LCP 개선: 히어로 이미지 우선 로딩 -->
<img
  src="/images/hero.webp"
  alt="히어로 이미지"
  fetchpriority="high"
  width="1200"
  height="600"
/>
```

### CLS (Cumulative Layout Shift)

사용자가 예상치 못한 레이아웃 이동을 경험하는 정도를 나타냅니다. 이미지나 광고가 늦게 로드되며 콘텐츠가 밀리는 현상이 대표적입니다.

- **좋음**: 0.1 이하
- **개선 필요**: 0.1~0.25
- **나쁨**: 0.25 초과

이미지와 비디오 요소에 반드시 `width`와 `height` 속성을 명시하거나 `aspect-ratio`를 CSS로 지정해야 합니다.

```css
/* CLS 방지: 이미지 컨테이너에 비율 고정 */
.image-wrapper {
  aspect-ratio: 16 / 9;
  overflow: hidden;
}

.image-wrapper img {
  width: 100%;
  height: 100%;
  object-fit: cover;
}
```

### INP (Interaction to Next Paint)

사용자의 모든 상호작용(클릭, 탭, 키보드 입력)에 대한 응답성을 측정합니다. 2024년 FID를 대체한 지표입니다.

- **좋음**: 200ms 이하
- **개선 필요**: 200~500ms
- **나쁨**: 500ms 초과

---

## 이미지 최적화

이미지는 웹페이지 용량의 50~70%를 차지합니다. 여기서 얻는 성과가 가장 큽니다.

### 차세대 포맷 사용

WebP와 AVIF는 JPEG, PNG 대비 20~50% 더 작은 파일 크기를 제공합니다.

```html
<picture>
  <source srcset="/images/photo.avif" type="image/avif" />
  <source srcset="/images/photo.webp" type="image/webp" />
  <img src="/images/photo.jpg" alt="사진" width="800" height="600" />
</picture>
```

### 지연 로딩(Lazy Loading) 적용

뷰포트 밖에 있는 이미지는 나중에 로딩하도록 설정합니다.

```html
<!-- 히어로 이미지는 eager, 나머지는 lazy -->
<img src="/images/content.webp" alt="콘텐츠" loading="lazy" />
```

### 반응형 이미지 제공

`srcset`으로 디바이스 해상도에 맞는 이미지를 제공합니다.

```html
<img
  src="/images/photo-800.webp"
  srcset="
    /images/photo-400.webp  400w,
    /images/photo-800.webp  800w,
    /images/photo-1200.webp 1200w
  "
  sizes="(max-width: 600px) 400px, (max-width: 900px) 800px, 1200px"
  alt="반응형 이미지"
/>
```

---

## JavaScript 번들 최적화

무거운 JavaScript 번들은 초기 렌더링을 차단하는 주범입니다.

### 코드 스플리팅

필요한 코드만 그때그때 로딩하는 방식입니다. React에서는 `React.lazy()`와 `Suspense`를 조합합니다.

```javascript
import React, { lazy, Suspense } from 'react';

// 번들을 분리하여 필요할 때만 로딩
const HeavyComponent = lazy(() => import('./HeavyComponent'));

function App() {
  return (
    <Suspense fallback={<div>로딩 중...</div>}>
      <HeavyComponent />
    </Suspense>
  );
}
```

### 트리 쉐이킹(Tree Shaking)

사용하지 않는 코드를 번들에서 제거합니다. 라이브러리를 임포트할 때 필요한 부분만 가져오는 습관이 중요합니다.

```javascript
// 나쁜 예: 전체 라이브러리 임포트
import _ from 'lodash';
const result = _.debounce(fn, 300);

// 좋은 예: 필요한 함수만 임포트 (번들 크기 대폭 감소)
import debounce from 'lodash/debounce';
const result = debounce(fn, 300);
```

### 렌더 블로킹 스크립트 제거

`defer`나 `async`를 사용해 스크립트가 HTML 파싱을 막지 않도록 합니다.

```html
<!-- defer: DOM 파싱 완료 후 순서대로 실행 -->
<script src="app.js" defer></script>

<!-- async: 다운로드 즉시 실행 (순서 보장 안 됨) -->
<script src="analytics.js" async></script>
```

---

## 폰트 최적화

웹폰트는 텍스트 렌더링을 지연시킬 수 있습니다.

```html
<!-- 폰트 파일 미리 로딩 -->
<link rel="preload" href="/fonts/MyFont.woff2" as="font" type="font/woff2" crossorigin />
```

```css
/* FOUT 방지: font-display 설정 */
@font-face {
  font-family: 'MyFont';
  src: url('/fonts/MyFont.woff2') format('woff2');
  font-display: swap; /* 시스템 폰트를 먼저 보여주고 교체 */
}
```

---

## 캐싱 전략

올바른 캐싱 설정만으로도 반복 방문 사용자의 경험이 크게 개선됩니다.

```
# Nginx 예시: 정적 자산 캐싱
location ~* \.(js|css|png|jpg|webp|woff2)$ {
    expires 1y;
    add_header Cache-Control "public, immutable";
}

# HTML 파일은 캐싱하지 않음
location ~* \.html$ {
    add_header Cache-Control "no-cache";
}
```

---

## 성능 측정 도구

최적화 전후 반드시 측정해야 개선 여부를 확인할 수 있습니다.

| 도구 | 용도 | 특징 |
|------|------|------|
| Lighthouse | 종합 성능 분석 | Chrome DevTools 내장 |
| PageSpeed Insights | 실사용 데이터 + 랩 데이터 | Google 공식 제공 |
| WebPageTest | 상세 워터폴 분석 | 다양한 지역·브라우저 테스트 |
| Chrome DevTools | 실시간 프로파일링 | 병목 구간 파악에 최적 |

---

## 실전 체크리스트

성능 최적화 작업을 시작할 때 활용하세요.

**이미지**
- [ ] WebP 또는 AVIF 포맷 사용
- [ ] `width`/`height` 속성 명시 (CLS 방지)
- [ ] 뷰포트 외 이미지에 `loading="lazy"` 적용
- [ ] `srcset`으로 반응형 이미지 제공

**JavaScript**
- [ ] 코드 스플리팅 적용
- [ ] 사용하지 않는 의존성 제거
- [ ] `defer`/`async` 속성 활용
- [ ] 번들 사이즈 정기 모니터링 (`bundlephobia` 활용)

**기타**
- [ ] 폰트 `preload` 및 `font-display: swap` 설정
- [ ] 정적 자산 캐싱 헤더 설정
- [ ] HTTP/2 또는 HTTP/3 활성화
- [ ] CDN 사용

---

웹 성능 최적화는 한 번에 끝나는 작업이 아닙니다. Lighthouse 점수를 주기적으로 모니터링하고, 새 기능을 추가할 때마다 성능 영향을 함께 검토하는 습관을 들이는 것이 중요합니다. 작은 개선이 쌓여 사용자 경험을 크게 향상시킵니다.
