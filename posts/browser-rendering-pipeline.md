---
title: 브라우저 렌더링 완전 가이드 — DOM부터 픽셀까지, 웹 페이지가 화면에 그려지는 원리
date: 2026-11-14
tags: [브라우저 렌더링, Critical Rendering Path, 웹 성능, DOM, 웹 개발]
description: HTML 파일을 받은 브라우저가 픽셀을 화면에 그리기까지의 렌더링 파이프라인 전 과정을 이해하고 성능 최적화에 적용하는 방법을 소개합니다.
---

웹 페이지를 열면 0.1초 안에 콘텐츠가 화면에 나타납니다. 당연해 보이지만 그 짧은 시간 안에 브라우저는 엄청난 작업을 수행합니다. HTML·CSS·JavaScript를 파싱하고, 트리를 구성하고, 레이아웃을 계산하고, 픽셀을 화면에 그립니다. 이 과정을 **Critical Rendering Path(핵심 렌더링 경로)**라고 부릅니다.

렌더링 파이프라인을 이해하면 단순히 "느리다"가 아니라 "어느 단계에서 왜 느린지"를 정확히 짚을 수 있습니다. 최적화는 원인을 아는 것에서 시작합니다.

---

## 렌더링 파이프라인 전체 흐름

브라우저가 화면을 그리는 과정은 크게 여섯 단계로 나뉩니다.

```
HTML/CSS/JS 수신
      ↓
  DOM 생성
      ↓
 CSSOM 생성
      ↓
 Render Tree 생성
      ↓
  Layout (Reflow)
      ↓
    Paint
      ↓
  Composite
```

각 단계를 순서대로 살펴보겠습니다.

---

## 1단계: DOM 생성

브라우저는 서버에서 받은 HTML 바이트 스트림을 다음 과정으로 변환합니다.

1. **바이트 → 문자**: `Content-Type` 헤더의 인코딩(UTF-8 등)에 따라 디코딩
2. **문자 → 토큰**: `<html>`, `<body>` 같은 태그를 토큰으로 분리
3. **토큰 → 노드**: 각 토큰을 속성과 규칙을 가진 노드 객체로 변환
4. **노드 → DOM**: 부모-자식 관계를 반영한 트리 구조로 조립

```html
<!-- 이 HTML이 -->
<html>
  <head><title>Hello</title></head>
  <body>
    <h1>제목</h1>
    <p>내용</p>
  </body>
</html>
```

```
Document
 └─ html
     ├─ head
     │   └─ title → "Hello"
     └─ body
         ├─ h1 → "제목"
         └─ p → "내용"
```

> **파싱 중단 주의**: `<script>` 태그를 만나면 DOM 파싱이 멈춥니다. JavaScript가 DOM을 변경할 수 있기 때문입니다. `defer`나 `async` 속성을 활용해 이 블로킹을 피할 수 있습니다.

---

## 2단계: CSSOM 생성

HTML 파싱 중 `<link rel="stylesheet">` 또는 `<style>` 태그를 만나면 CSS를 파싱해 **CSSOM(CSS Object Model)**을 구성합니다. 과정은 DOM 생성과 유사합니다.

```css
body { font-size: 16px; }
h1   { color: #333; font-size: 2em; }
p    { color: #666; }
```

```
StyleSheet
 ├─ body → { font-size: 16px }
 ├─ h1   → { color: #333, font-size: 32px }  /* 2em = 32px */
 └─ p    → { color: #666 }
```

CSSOM은 **렌더링 차단 리소스**입니다. 브라우저는 모든 CSS를 처리하기 전까지 Render Tree를 만들지 않습니다. 스타일 일부만 적용된 화면이 잠깐 보이는 **FOUC(Flash of Unstyled Content)** 현상을 막기 위해서입니다.

---

## 3단계: Render Tree 생성

DOM과 CSSOM을 합쳐 **Render Tree**를 만듭니다. Render Tree에는 화면에 실제로 표시될 노드만 포함됩니다.

- `display: none` 요소 → Render Tree에서 제외
- `<head>`, `<script>`, `<meta>` 등 → 제외
- `visibility: hidden` 요소 → **포함** (공간은 차지하되 보이지 않음)

```
RenderTree
 ├─ body
 │   ├─ h1 (font-size: 32px, color: #333)
 │   └─ p  (font-size: 16px, color: #666)
 └─ ...
```

---

## 4단계: Layout (Reflow)

Render Tree가 준비되면 각 노드의 **정확한 위치와 크기**를 계산합니다. 이 과정을 **Layout** 또는 **Reflow**라고 부릅니다.

- `%`, `em`, `vw` 같은 상대 단위를 픽셀로 변환
- Flexbox, Grid 알고리즘을 실행해 자식 요소 배치 계산
- 뷰포트 크기를 기준으로 모든 요소의 `x, y, width, height` 결정

Layout은 **비용이 큰 작업**입니다. 한 요소의 크기 변경이 주변 요소에 영향을 미치기 때문에 DOM 전체를 다시 계산해야 할 수 있습니다.

```javascript
// Reflow를 유발하는 속성 읽기/쓰기 (피해야 할 패턴)
const width = element.offsetWidth;  // 최신 Layout 값 강제 계산
element.style.width = width + 10 + 'px';  // Reflow 발생
const newWidth = element.offsetWidth;  // 또 강제 계산 — Forced Reflow!
```

---

## 5단계: Paint

Layout이 끝나면 각 노드를 실제 픽셀로 그립니다. 텍스트, 색상, 이미지, 테두리, 그림자 같은 **시각적 속성**을 처리합니다.

브라우저는 성능을 위해 페이지를 여러 **레이어**로 분리해 그립니다. 레이어 분리 기준:

- `position: fixed/sticky` 요소
- `transform`, `opacity` 애니메이션이 적용된 요소
- `will-change` 속성이 지정된 요소
- `<video>`, `<canvas>`, `<iframe>`

---

## 6단계: Composite

각 레이어를 GPU에서 하나의 화면으로 합성합니다. GPU는 이 작업을 하드웨어 가속으로 처리하기 때문에 매우 빠릅니다.

`transform`과 `opacity` 변경은 **Layout과 Paint를 건너뛰고** Composite 단계만 거칩니다. 그래서 애니메이션에서 이 두 속성을 사용하면 성능이 좋습니다.

```css
/* 느린 방법: Layout → Paint → Composite */
.box { transition: left 0.3s; }
.box:hover { left: 100px; }

/* 빠른 방법: Composite만 */
.box { transition: transform 0.3s; }
.box:hover { transform: translateX(100px); }
```

---

## Reflow vs Repaint — 무엇이 더 비쌀까?

| 속성 변경 | Reflow | Repaint | Composite |
|-----------|--------|---------|-----------|
| `width`, `height`, `margin` | ✅ | ✅ | ✅ |
| `color`, `background` | ❌ | ✅ | ✅ |
| `transform`, `opacity` | ❌ | ❌ | ✅ |

Reflow > Repaint > Composite 순으로 비용이 큽니다. 가능하면 `transform`과 `opacity`만 건드리는 애니메이션을 설계하세요.

---

## 실전 최적화 패턴

### DOM 읽기·쓰기 분리 (Layout Thrashing 방지)

```javascript
// 나쁜 예: 읽기와 쓰기가 번갈아 Forced Reflow 유발
elements.forEach(el => {
  const h = el.offsetHeight;        // 읽기 → Reflow 강제
  el.style.height = h * 2 + 'px';  // 쓰기
});

// 좋은 예: 먼저 모두 읽고, 그 다음 모두 쓰기
const heights = elements.map(el => el.offsetHeight);  // 읽기만
elements.forEach((el, i) => {
  el.style.height = heights[i] * 2 + 'px';           // 쓰기만
});
```

### `will-change`로 레이어 미리 분리

```css
.animated-card {
  will-change: transform;  /* GPU 레이어 미리 생성 */
}
```

남용하면 오히려 메모리를 낭비하므로 실제로 애니메이션이 발생하는 요소에만 사용하세요.

### `requestAnimationFrame`으로 애니메이션 동기화

```javascript
function animate() {
  element.style.transform = `translateX(${pos}px)`;
  pos += 1;
  requestAnimationFrame(animate);  // 다음 프레임에 실행 예약
}
requestAnimationFrame(animate);
```

`rAF`는 브라우저의 렌더링 사이클에 맞춰 콜백을 실행해 불필요한 Reflow를 방지합니다.

---

## DevTools로 렌더링 병목 찾기

Chrome DevTools의 **Performance** 탭을 열고 녹화하면 각 프레임에서 어떤 단계가 얼마나 걸렸는지 확인할 수 있습니다.

1. `F12` → Performance 탭 → 🔴 Record
2. 느린 인터랙션 재현
3. 녹화 중지 후 타임라인 분석

색상 코드:
- 🟡 **노란색**: JavaScript 실행
- 🟣 **보라색**: Layout/Reflow
- 🟢 **초록색**: Paint
- 🔵 **파란색**: Composite

긴 보라색 블록이 보이면 Reflow 최적화가 필요하다는 신호입니다.

---

## 정리

브라우저 렌더링 파이프라인을 요약하면 다음과 같습니다.

1. **DOM** — HTML을 트리로 파싱
2. **CSSOM** — CSS를 트리로 파싱 (렌더링 차단)
3. **Render Tree** — 표시할 노드만 선별
4. **Layout** — 위치·크기 계산 (비쌈)
5. **Paint** — 픽셀로 그리기
6. **Composite** — 레이어 합성 (GPU, 빠름)

성능 최적화의 핵심은 **비싼 단계를 건너뛰는 것**입니다. Layout을 유발하는 속성 변경을 줄이고, 애니메이션은 `transform`과 `opacity`로만 처리하고, DOM 읽기·쓰기를 분리하면 60fps의 부드러운 UI를 구현할 수 있습니다.
