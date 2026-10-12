---
title: PWA(프로그레시브 웹 앱) 입문 가이드 — 웹 앱을 네이티브처럼 만드는 기술
date: 2026-10-12
tags: [PWA, 웹 개발, Service Worker, 오프라인, 프론트엔드]
description: Service Worker와 Web App Manifest를 활용해 오프라인 지원, 푸시 알림, 홈 화면 설치까지 가능한 PWA를 구현하는 방법을 소개합니다.
---

스마트폰 앱을 따로 설치하지 않아도 네이티브 앱처럼 동작하는 웹 사이트를 경험해 본 적 있으신가요? 그것이 바로 **PWA(Progressive Web App)** 입니다. 트위터, 스타벅스, 핀터레스트 같은 서비스들이 PWA를 적극 도입하면서 웹 앱과 네이티브 앱의 경계가 빠르게 허물어지고 있습니다. 이 글에서는 PWA의 핵심 기술을 이해하고 직접 구현하는 방법을 단계별로 정리합니다.

## PWA란 무엇인가

PWA는 웹 기술(HTML, CSS, JavaScript)로 만들어진 앱이지만, 네이티브 앱에 가까운 사용자 경험을 제공하는 웹 앱입니다. 핵심 특징은 세 가지입니다.

- **신뢰성**: 네트워크 연결이 불안정하거나 오프라인 상태에서도 동작합니다.
- **빠른 속도**: 캐싱된 리소스를 활용해 즉각적으로 반응합니다.
- **설치 가능**: 홈 화면에 추가하면 독립 앱처럼 실행됩니다.

이 세 가지를 실현하는 기술이 **Service Worker**, **Web App Manifest**, **HTTPS**입니다.

## Service Worker — PWA의 심장

Service Worker는 브라우저 백그라운드에서 동작하는 JavaScript 파일입니다. 웹 페이지와 별개의 스레드에서 실행되기 때문에 페이지가 닫혀 있어도 네트워크 요청을 가로채고 캐싱을 처리할 수 있습니다.

### Service Worker 등록

```javascript
// main.js
if ('serviceWorker' in navigator) {
  window.addEventListener('load', () => {
    navigator.serviceWorker
      .register('/sw.js')
      .then(reg => console.log('SW 등록 완료:', reg.scope))
      .catch(err => console.error('SW 등록 실패:', err));
  });
}
```

### Service Worker 생명주기

Service Worker는 **install → activate → fetch** 세 단계를 거칩니다.

```javascript
// sw.js
const CACHE_NAME = 'my-app-v1';
const URLS_TO_CACHE = ['/', '/index.html', '/styles.css', '/app.js'];

// 설치 단계: 핵심 리소스 캐싱
self.addEventListener('install', event => {
  event.waitUntil(
    caches.open(CACHE_NAME).then(cache => cache.addAll(URLS_TO_CACHE))
  );
});

// 활성화 단계: 이전 캐시 정리
self.addEventListener('activate', event => {
  event.waitUntil(
    caches.keys().then(keys =>
      Promise.all(
        keys
          .filter(key => key !== CACHE_NAME)
          .map(key => caches.delete(key))
      )
    )
  );
});
```

## 오프라인 캐싱 전략

캐싱 전략은 앱의 성격에 따라 다르게 적용해야 합니다. 자주 사용되는 세 가지 패턴을 소개합니다.

### 1. Cache First (캐시 우선)

정적 리소스처럼 자주 바뀌지 않는 파일에 적합합니다.

```javascript
self.addEventListener('fetch', event => {
  event.respondWith(
    caches.match(event.request).then(cached => {
      return cached || fetch(event.request);
    })
  );
});
```

### 2. Network First (네트워크 우선)

최신 데이터가 중요한 API 요청에 적합합니다. 네트워크 실패 시 캐시를 폴백으로 사용합니다.

```javascript
self.addEventListener('fetch', event => {
  event.respondWith(
    fetch(event.request)
      .then(response => {
        const clone = response.clone();
        caches.open(CACHE_NAME).then(cache => cache.put(event.request, clone));
        return response;
      })
      .catch(() => caches.match(event.request))
  );
});
```

### 3. Stale While Revalidate

캐시를 즉시 반환하면서 백그라운드에서 갱신합니다. 속도와 최신성을 동시에 잡는 패턴입니다.

```javascript
self.addEventListener('fetch', event => {
  event.respondWith(
    caches.open(CACHE_NAME).then(cache =>
      cache.match(event.request).then(cached => {
        const networkFetch = fetch(event.request).then(response => {
          cache.put(event.request, response.clone());
          return response;
        });
        return cached || networkFetch;
      })
    )
  );
});
```

## Web App Manifest — 설치 경험 설정

`manifest.json` 파일로 앱 이름, 아이콘, 테마 색상, 실행 모드 등을 정의합니다.

```json
{
  "name": "My Progressive App",
  "short_name": "MyApp",
  "description": "PWA 예제 애플리케이션",
  "start_url": "/",
  "display": "standalone",
  "background_color": "#ffffff",
  "theme_color": "#3b82f6",
  "icons": [
    {
      "src": "/icons/icon-192.png",
      "sizes": "192x192",
      "type": "image/png"
    },
    {
      "src": "/icons/icon-512.png",
      "sizes": "512x512",
      "type": "image/png"
    }
  ]
}
```

HTML에서 manifest를 연결합니다.

```html
<link rel="manifest" href="/manifest.json" />
<meta name="theme-color" content="#3b82f6" />
```

`display` 값에 따라 실행 방식이 달라집니다.

| 값 | 설명 |
|---|---|
| `standalone` | 네이티브 앱처럼 브라우저 UI 없이 실행 |
| `fullscreen` | 상태 표시줄까지 숨기는 전체 화면 |
| `minimal-ui` | 최소한의 브라우저 UI 유지 |
| `browser` | 일반 브라우저 탭으로 실행 |

## Lighthouse로 PWA 점수 확인하기

Chrome DevTools의 **Lighthouse** 탭에서 PWA 체크리스트를 바로 확인할 수 있습니다. 주요 항목은 다음과 같습니다.

- [ ] HTTPS로 서비스 중
- [ ] Service Worker 등록 및 오프라인 응답
- [ ] 유효한 `manifest.json` 존재
- [ ] 192×192 이상의 아이콘 제공
- [ ] `viewport` 메타 태그 설정
- [ ] 3G 환경에서 3초 이내 로드

## 실무 도입 시 고려사항

PWA를 실무에 적용할 때는 몇 가지를 주의해야 합니다.

**캐시 버전 관리**: Service Worker를 업데이트할 때 `CACHE_NAME`의 버전을 올려야 사용자에게 최신 앱이 전달됩니다.

**iOS 제한**: Safari는 Service Worker를 지원하지만 푸시 알림 등 일부 API는 제한적입니다. iOS 사용자 비율이 높다면 사전에 호환성을 검토하세요.

**업데이트 알림**: 새 Service Worker가 배포됐을 때 사용자에게 "새 버전이 있습니다. 새로고침 하시겠습니까?" 같은 UI를 제공하면 부드러운 업데이트 경험을 줄 수 있습니다.

## 마치며

PWA는 추가 설치 없이 웹을 통해 네이티브 수준의 경험을 제공할 수 있는 강력한 기술입니다. Service Worker로 오프라인 지원과 성능을 챙기고, Web App Manifest로 홈 화면 설치 경험을 완성하면 사용자 유지율을 크게 높일 수 있습니다. 기존 웹 프로젝트에 점진적으로(Progressive) 적용할 수 있다는 것도 PWA의 큰 장점입니다. 오늘 Lighthouse로 현재 프로젝트의 PWA 점수를 측정해 보는 것부터 시작해 보세요.
