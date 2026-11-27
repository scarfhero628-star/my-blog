---
title: 웹 폰트 최적화 완전 가이드 — 로딩 속도와 사용자 경험을 동시에 잡는 실전 전략
date: 2026-11-27
tags: [웹 폰트, 성능, CSS, 웹 개발, UX]
description: 웹 폰트가 야기하는 성능 문제를 이해하고 FOUT·FOIT를 방지하며 로딩 속도를 극대화하는 실전 최적화 전략을 소개합니다.
---

## 왜 웹 폰트 최적화가 중요한가

웹 폰트는 사이트에 개성과 브랜드 정체성을 부여하는 핵심 요소입니다. 하지만 최적화 없이 무분별하게 사용하면 페이지 로딩 속도를 크게 떨어뜨리고 사용자 경험(UX)에 직접적인 타격을 줍니다. Google Fonts 같은 외부 폰트를 그냥 `<link>`로 불러오기만 해도 렌더링 차단, 추가 HTTP 요청, 큰 파일 용량이라는 세 가지 문제가 동시에 발생합니다.

특히 Core Web Vitals에서 LCP(Largest Contentful Paint)와 CLS(Cumulative Layout Shift)는 폰트 로딩과 직결됩니다. 폰트가 늦게 도착할 경우 텍스트가 갑자기 바뀌며 레이아웃이 흔들리는 현상이 CLS 점수를 끌어내립니다.

---

## FOUT와 FOIT — 폰트 로딩의 두 가지 악몽

웹 폰트가 완전히 로드되기 전까지 브라우저가 텍스트를 어떻게 처리하는지에 따라 두 가지 현상이 나타납니다.

### FOUT (Flash of Unstyled Text)

폰트가 로드되기 전에 시스템 폰트로 텍스트를 먼저 보여준 뒤, 웹 폰트가 도착하면 교체하는 방식입니다. Firefox와 IE의 기본 동작이며, 텍스트가 갑자기 모양이 바뀌는 깜박임 현상이 생깁니다.

### FOIT (Flash of Invisible Text)

폰트가 로드되기 전까지 텍스트를 아예 숨기는 방식입니다. Chrome과 Safari의 기본 동작으로, 보통 최대 3초 동안 텍스트가 보이지 않습니다. 느린 네트워크 환경에서는 사용자가 콘텐츠를 전혀 읽지 못하는 최악의 상황이 발생할 수 있습니다.

---

## `font-display` — 가장 간단한 해결책

CSS `@font-face`의 `font-display` 속성 하나로 FOUT/FOIT를 직접 제어할 수 있습니다.

```css
@font-face {
  font-family: 'MyFont';
  src: url('/fonts/myfont.woff2') format('woff2');
  font-display: swap; /* 권장 */
}
```

| 값 | 동작 | 적합한 상황 |
|---|---|---|
| `auto` | 브라우저 기본값 | - |
| `block` | FOIT (최대 3초 차단) | 아이콘 폰트 |
| `swap` | 즉시 fallback 표시 후 교체 | 본문 텍스트 |
| `fallback` | 100ms 차단 후 swap | 균형이 필요할 때 |
| `optional` | 짧은 차단, 로드 실패 시 포기 | 장식용 폰트 |

대부분의 본문 텍스트에는 **`font-display: swap`** 을 사용하는 것이 권장됩니다.

---

## 폰트 파일 포맷 최적화

폰트 파일 포맷 선택만으로도 파일 크기를 획기적으로 줄일 수 있습니다.

### WOFF2를 우선적으로 사용하세요

```css
@font-face {
  font-family: 'MyFont';
  src:
    url('/fonts/myfont.woff2') format('woff2'),
    url('/fonts/myfont.woff') format('woff');
}
```

WOFF2는 Brotli 압축을 사용하여 WOFF 대비 약 30% 더 작습니다. 현재 모든 모던 브라우저가 지원하므로 WOFF2 하나만 제공해도 대부분의 환경을 커버할 수 있습니다.

---

## 서브세팅 — 필요한 글자만 담기

특히 한국어 폰트는 수만 개의 글자를 포함해 파일 크기가 수 MB에 달하는 경우가 많습니다. **서브세팅(Subsetting)** 을 통해 실제로 사용하는 문자만 추출하면 파일 크기를 90% 이상 줄일 수 있습니다.

```bash
# pyftsubset 도구로 서브세팅
pyftsubset NotoSansKR-Regular.ttf \
  --text-file=used-chars.txt \
  --flavor=woff2 \
  --output-file=NotoSansKR-subset.woff2
```

Google Fonts의 `text` 파라미터를 사용하면 URL 단계에서 서브세팅할 수도 있습니다.

```html
<link href="https://fonts.googleapis.com/css2?family=Noto+Sans+KR&text=안녕하세요" rel="stylesheet">
```

---

## 폰트 프리로딩 — 브라우저에게 먼저 힌트 주기

`<link rel="preload">`를 사용하면 브라우저가 HTML을 파싱하면서 폰트 다운로드를 미리 시작합니다.

```html
<link
  rel="preload"
  href="/fonts/myfont.woff2"
  as="font"
  type="font/woff2"
  crossorigin
/>
```

> **주의:** `crossorigin` 속성은 같은 도메인의 폰트라도 반드시 넣어야 합니다. 없으면 폰트가 두 번 다운로드됩니다.

---

## 폰트 자가 호스팅 vs CDN

| 방식 | 장점 | 단점 |
|---|---|---|
| Google Fonts | 간편함, 자동 최적화 | 추가 DNS 조회, 개인정보 이슈 |
| 자가 호스팅 | 완전한 제어, 프리로드 가능 | 초기 설정 필요 |

구글 Fonts를 자가 호스팅으로 전환하려면 [google-webfonts-helper](https://gwfh.mranftl.com) 도구를 활용하면 편리합니다. 필요한 폰트를 선택하면 CSS와 파일을 한꺼번에 받을 수 있습니다.

---

## 실전 체크리스트

최적화 작업을 마무리하기 전에 아래 항목을 점검하세요.

- [ ] WOFF2 포맷 사용
- [ ] `font-display: swap` 적용
- [ ] 중요 폰트에 `<link rel="preload">` 추가
- [ ] 사용하지 않는 굵기(weight)와 스타일 제거
- [ ] 한국어 폰트는 서브세팅 적용
- [ ] Chrome DevTools Network 탭에서 폰트 요청 수 확인
- [ ] Lighthouse로 `font-display` 경고 확인

---

## 마치며

웹 폰트 최적화는 복잡해 보이지만, `font-display: swap` 적용과 WOFF2 포맷 사용만으로도 체감할 수 있는 개선이 가능합니다. 한국어 폰트처럼 파일이 큰 경우에는 서브세팅이 가장 효과적인 단일 최적화 기법입니다. 작은 설정 하나가 사용자의 첫인상을 결정짓는 로딩 경험을 크게 바꿀 수 있습니다.
