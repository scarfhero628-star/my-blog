---
title: 웹 접근성 기초 — WCAG로 더 포용적인 웹 만들기
date: 2026-09-22
tags: [웹 접근성, A11y, WCAG, 웹 개발, HTML]
description: WCAG 원칙을 바탕으로 더 많은 사용자가 이용할 수 있는 접근성 높은 웹 페이지를 만드는 방법을 소개합니다.
---

## 왜 웹 접근성인가

웹은 '모두를 위한 공간'으로 설계되었습니다. 팀 버너스리가 WWW를 만들 때의 핵심 철학은 장애 여부, 기기 종류, 언어와 무관하게 누구나 정보에 접근할 수 있어야 한다는 것이었습니다. 하지만 현실에서는 시각 장애, 청각 장애, 운동 장애, 인지 장애를 가진 수억 명의 사용자가 여전히 수많은 웹사이트에서 불편함을 겪습니다.

웹 접근성(Web Accessibility, A11y)은 단순한 법적 의무나 CSR 활동이 아닙니다. 접근성이 좋은 웹사이트는 더 나은 SEO, 더 넓은 사용자층, 유지보수가 쉬운 코드라는 혜택을 함께 가져옵니다.

## WCAG란 무엇인가

WCAG(Web Content Accessibility Guidelines)는 W3C가 제정한 웹 접근성 표준입니다. 현재 2.1 버전이 가장 널리 사용되며, 2.2와 3.0이 개발 중입니다.

WCAG는 네 가지 핵심 원칙(POUR)으로 구성됩니다.

### POUR 원칙

- **P — Perceivable(인식 가능)**: 텍스트 대체 콘텐츠, 자막, 색상 외 다른 정보 전달 수단
- **O — Operable(조작 가능)**: 키보드만으로 모든 기능 사용 가능, 충분한 시간 제공
- **U — Understandable(이해 가능)**: 읽기 쉬운 콘텐츠, 예측 가능한 UI, 오류 도움말
- **R — Robust(견고)**: 다양한 보조 기술과 호환 가능한 코드

레벨은 A, AA, AAA 세 단계로 나뉩니다. 대부분의 법적 기준은 AA 준수를 요구합니다.

## 시맨틱 HTML이 전부다

접근성의 기초는 시맨틱 HTML에 있습니다. `<div>`와 `<span>`으로만 구성된 페이지는 스크린 리더가 구조를 파악하기 어렵습니다.

### 나쁜 예

```html
<div class="header">내 블로그</div>
<div class="nav">
  <div class="nav-item">홈</div>
  <div class="nav-item">소개</div>
</div>
<div class="main">
  <div class="article-title">첫 번째 포스트</div>
  <div class="article-content">...</div>
</div>
```

### 좋은 예

```html
<header>
  <h1>내 블로그</h1>
</header>
<nav aria-label="주 메뉴">
  <ul>
    <li><a href="/">홈</a></li>
    <li><a href="/about">소개</a></li>
  </ul>
</nav>
<main>
  <article>
    <h2>첫 번째 포스트</h2>
    <p>...</p>
  </article>
</main>
```

스크린 리더 사용자는 `header`, `nav`, `main` 같은 랜드마크를 통해 빠르게 페이지를 탐색합니다. `<h1>` ~ `<h6>`의 올바른 계층 구조도 탐색 효율을 크게 높입니다.

## 이미지 대체 텍스트

모든 의미 있는 이미지에는 `alt` 속성이 필요합니다.

```html
<!-- 좋은 예: 이미지가 정보를 전달할 때 -->
<img src="chart.png" alt="2024년 1분기 매출: 전년 대비 23% 증가" />

<!-- 좋은 예: 장식용 이미지는 alt를 비워두기 -->
<img src="divider.svg" alt="" role="presentation" />

<!-- 나쁜 예: 무의미한 alt -->
<img src="chart.png" alt="이미지" />
```

SVG 아이콘에도 접근성을 챙겨야 합니다.

```html
<!-- 버튼에 아이콘만 있는 경우 -->
<button aria-label="검색">
  <svg aria-hidden="true" focusable="false">...</svg>
</button>
```

## 색상과 명도 대비

색상만으로 정보를 전달해서는 안 됩니다. "빨간색이 오류입니다"라는 표현은 색맹 사용자에게 통하지 않습니다.

WCAG AA 기준의 색상 명도 대비는 다음과 같습니다.

| 요소 | 최소 대비율 |
|------|------------|
| 일반 텍스트 | 4.5:1 |
| 큰 텍스트 (18pt 이상) | 3:1 |
| UI 컴포넌트, 아이콘 | 3:1 |

```css
/* 명도 대비 부족 (NG) */
.error { color: #ff6666; background: #ffffff; } /* 대비율 약 3.1:1 */

/* 명도 대비 충분 (OK) */
.error { color: #cc0000; background: #ffffff; } /* 대비율 약 5.9:1 */
```

## 키보드 접근성

많은 사용자가 마우스 없이 키보드만으로 웹을 탐색합니다. 모든 인터랙티브 요소는 키보드로 접근 가능해야 합니다.

### 포커스 관리

```css
/* 포커스 스타일을 절대 지우지 말 것 */

/* 나쁜 예 */
:focus { outline: none; }

/* 좋은 예: 더 보기 좋게 커스텀 */
:focus-visible {
  outline: 2px solid #0066cc;
  outline-offset: 2px;
  border-radius: 2px;
}
```

모달이나 드롭다운이 열릴 때는 포커스를 해당 컴포넌트 안으로 이동시키고, 닫힐 때는 원래 트리거 요소로 돌려보내야 합니다.

## ARIA 속성 활용

ARIA(Accessible Rich Internet Applications)는 HTML만으로 표현하기 어려운 역할, 상태, 속성을 보조 기술에 전달합니다.

```html
<!-- 토글 버튼 -->
<button
  aria-pressed="false"
  aria-label="알림 켜기/끄기"
  id="notif-toggle">
  🔔
</button>

<!-- 아코디언 패널 -->
<button
  aria-expanded="false"
  aria-controls="panel-1">
  상세 정보 보기
</button>
<div id="panel-1" hidden>
  <p>상세 내용...</p>
</div>

<!-- 진행률 표시 -->
<div
  role="progressbar"
  aria-valuenow="60"
  aria-valuemin="0"
  aria-valuemax="100"
  aria-label="업로드 중">
  <div style="width: 60%"></div>
</div>
```

**ARIA를 사용할 때 첫 번째 규칙**: 네이티브 HTML 요소로 할 수 있다면 ARIA 대신 HTML을 쓰세요. `<button>`이 `<div role="button">`보다 항상 낫습니다.

## 접근성 테스트 도구

| 도구 | 용도 |
|------|------|
| axe DevTools | Chrome 확장 프로그램, 자동화 테스트 |
| Lighthouse | Chrome DevTools 내장 접근성 감사 |
| NVDA / JAWS | Windows 스크린 리더 |
| VoiceOver | macOS/iOS 내장 스크린 리더 |
| TalkBack | Android 내장 스크린 리더 |
| Color Contrast Checker | 명도 대비 측정 |

자동화 도구는 약 30~40%의 접근성 이슈만 감지합니다. 실제 스크린 리더로 직접 탐색해보는 것이 필수입니다.

## 흔한 접근성 실수

1. **`alt` 누락** — 의미 있는 이미지에 alt 없음
2. **포커스 스타일 제거** — `outline: none`으로 시각적 포커스 제거
3. **클릭 가능 `<div>`** — `<button>` 대신 클릭 이벤트 달린 `<div>` 사용
4. **자동 재생 미디어** — 소리 있는 영상 자동 재생
5. **낮은 색상 대비** — 트렌디한 파스텔/라이트 UI
6. **폼 레이블 누락** — `placeholder`만 있고 `<label>` 없음
7. **시간 제한** — 충분한 시간을 주지 않거나 연장 수단 없음

## 마치며

웹 접근성은 "장애인을 위한 특별한 배려"가 아닙니다. 잘 만들어진 접근성은 **노령 사용자, 저사양 기기 사용자, 이동 중인 사용자, 일시적인 부상을 당한 사용자** 모두에게 더 나은 경험을 제공합니다.

접근성은 처음부터 고려할수록 비용이 줄어듭니다. 오늘 당장 새 컴포넌트 하나를 만들 때 시맨틱 HTML과 키보드 접근성부터 시작해 보세요. 작은 변화가 수백만 명의 사용자 경험을 바꿀 수 있습니다.
