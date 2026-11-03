---
title: HTTP 캐싱 완전 가이드 — 브라우저와 서버 사이의 캐시 전략 마스터하기
date: 2026-11-03
tags: [HTTP, 캐싱, 웹 성능, Cache-Control, 웹 개발]
description: Cache-Control, ETag, Last-Modified 등 HTTP 캐싱 메커니즘을 이해하고 실무에서 바로 적용할 수 있는 효과적인 캐싱 전략을 소개합니다.
---

웹 성능 최적화에서 가장 강력한 도구 중 하나는 **HTTP 캐싱**입니다. 서버에 불필요한 요청을 보내지 않고 브라우저나 중간 프록시에 저장된 응답을 재사용하면, 페이지 로드 속도를 극적으로 개선하고 서버 부하를 줄일 수 있습니다. 그러나 캐싱은 잘못 설정하면 사용자가 오래된 데이터를 보게 되는 심각한 문제로 이어집니다. 이 글에서는 HTTP 캐싱의 핵심 메커니즘과 실무 전략을 체계적으로 정리합니다.

## HTTP 캐싱이란?

HTTP 캐싱은 이전에 가져온 리소스를 저장하고, 이후 요청 시 서버에 다시 요청하는 대신 저장된 복사본을 사용하는 기술입니다. 캐시는 크게 두 종류로 나뉩니다.

- **브라우저 캐시 (Private Cache)**: 사용자의 브라우저에만 저장되며, 해당 사용자만 사용합니다.
- **공유 캐시 (Shared Cache)**: CDN, 리버스 프록시 등 여러 사용자가 공유하는 캐시입니다.

캐시가 올바르게 설정되면 동일한 리소스에 대한 반복 요청을 서버까지 보내지 않고 즉시 응답할 수 있습니다.

## Cache-Control 헤더

`Cache-Control`은 HTTP/1.1에서 도입된 캐싱 제어의 핵심 헤더입니다. 서버 응답에 포함시켜 캐시 동작 방식을 정밀하게 제어할 수 있습니다.

### 주요 디렉티브

| 디렉티브 | 설명 |
|---|---|
| `max-age=<초>` | 지정한 초 동안 캐시를 신선한 상태로 유지합니다. |
| `no-cache` | 캐시를 저장하지만, 사용 전 반드시 서버에 재검증을 요청합니다. |
| `no-store` | 캐시를 전혀 저장하지 않습니다. 민감한 데이터에 사용합니다. |
| `public` | 공유 캐시(CDN 등)에 저장될 수 있습니다. |
| `private` | 브라우저 캐시에만 저장되고, 공유 캐시에는 저장되지 않습니다. |
| `immutable` | 리소스가 절대 변경되지 않음을 선언합니다. |
| `stale-while-revalidate=<초>` | 캐시가 만료된 후에도 지정한 초 동안 오래된 응답을 반환하면서 백그라운드에서 재검증합니다. |

```http
# 정적 에셋: 1년간 캐시, 변경되지 않음
Cache-Control: public, max-age=31536000, immutable

# HTML 파일: 항상 재검증
Cache-Control: no-cache

# 민감한 데이터: 캐시 금지
Cache-Control: no-store, private
```

## ETag와 조건부 요청

캐시가 만료된 후에도 리소스가 실제로 변경되지 않았다면 서버는 전체 내용을 다시 보내는 대신 `304 Not Modified`를 반환해 대역폭을 절약할 수 있습니다. 이를 위해 **조건부 요청**을 사용합니다.

### ETag

`ETag`는 리소스의 특정 버전을 나타내는 고유 식별자(주로 해시값)입니다.

```http
# 서버 초기 응답
HTTP/1.1 200 OK
ETag: "abc123"
Cache-Control: no-cache

# 브라우저 재요청 (ETag 포함)
GET /api/data HTTP/1.1
If-None-Match: "abc123"

# 리소스가 변경되지 않았을 때 서버 응답
HTTP/1.1 304 Not Modified
```

### Last-Modified

`Last-Modified`는 리소스의 마지막 수정 날짜를 나타냅니다. `ETag`보다 정밀도가 낮지만 여전히 널리 사용됩니다.

```http
# 서버 초기 응답
Last-Modified: Mon, 03 Nov 2026 09:00:00 GMT

# 브라우저 재요청
If-Modified-Since: Mon, 03 Nov 2026 09:00:00 GMT
```

ETag와 Last-Modified를 함께 제공하면 클라이언트는 둘 다 활용할 수 있습니다.

## 실전 캐싱 전략

리소스 종류에 따라 전략을 다르게 적용해야 합니다.

### 1. 해시된 정적 에셋 (JS, CSS, 이미지)

Vite, Webpack 등 번들러는 파일 내용이 바뀌면 파일명에 해시를 삽입합니다(`app.3f8a1b.js`). 파일명 자체가 버전이므로 `immutable`로 영구 캐시해도 안전합니다.

```nginx
# Nginx 설정 예시
location ~* \.(js|css|png|jpg|woff2)$ {
    add_header Cache-Control "public, max-age=31536000, immutable";
}
```

### 2. HTML 파일

HTML은 새 에셋을 참조하는 진입점이므로 항상 최신 버전을 확인해야 합니다.

```nginx
location ~* \.html$ {
    add_header Cache-Control "no-cache";
}
```

### 3. API 응답

데이터의 특성에 따라 다릅니다. 자주 변하지 않는 목록 데이터는 짧은 `max-age`와 `stale-while-revalidate`를 조합하면 빠른 응답과 데이터 신선도를 동시에 얻을 수 있습니다.

```http
# 60초 캐시, 이후 10분간 오래된 데이터 허용 + 백그라운드 재검증
Cache-Control: public, max-age=60, stale-while-revalidate=600
```

### 4. 인증이 필요한 응답

인증된 사용자의 데이터는 공유 캐시에 저장되면 안 됩니다.

```http
Cache-Control: private, no-cache
```

## 캐시 무효화 (Cache Busting)

`max-age`가 설정된 리소스를 강제로 무효화하려면 URL을 변경하는 것이 가장 확실한 방법입니다.

- **파일명 해시**: `main.abc123.js` → `main.def456.js`
- **쿼리 파라미터**: `/style.css?v=2`
- **버전 경로**: `/v2/api/data`

CDN을 사용한다면 CDN의 캐시 퍼지(Purge) API를 활용해 특정 리소스를 즉시 무효화할 수도 있습니다.

## 캐시 상태 확인하기

크롬 DevTools의 **Network 탭**에서 각 요청의 캐시 상태를 확인할 수 있습니다.

- **(from memory cache)**: 메모리에서 즉시 응답
- **(from disk cache)**: 디스크 캐시에서 응답
- **304**: 조건부 요청 성공, 내용 없이 캐시 재사용
- **200**: 실제 서버 응답

`Disable cache` 옵션을 해제하고 테스트해야 실제 사용자 경험과 동일한 동작을 확인할 수 있습니다.

## 마치며

HTTP 캐싱은 설정 몇 줄로 웹 성능을 크게 향상시킬 수 있는 강력한 도구입니다. 핵심은 **"무엇을 얼마나 오래 캐시할 수 있는가"**를 리소스 종류별로 명확히 파악하는 것입니다. 해시 기반 정적 에셋은 영구 캐시, HTML은 항상 재검증, API는 데이터 특성에 맞게 조정한다는 원칙을 기억해 두세요. 제대로 설계된 캐싱 전략 하나가 수많은 최적화 작업을 대체할 수 있습니다.
