---
title: CSS 커스텀 프로퍼티 완벽 가이드 — 변수로 디자인 시스템을 한 단계 업그레이드하기
date: 2026-09-26
tags: [CSS, CSS 변수, 커스텀 프로퍼티, 웹 개발, 디자인 시스템]
description: CSS 커스텀 프로퍼티(변수)의 기본 개념부터 다크모드·테마 시스템 구현까지, 실무에서 바로 활용할 수 있는 핵심 기법을 소개합니다.
---

CSS 커스텀 프로퍼티(Custom Properties), 흔히 **CSS 변수**라고 불리는 이 기능은 현대 CSS의 게임체인저입니다. Sass나 Less 같은 전처리기 없이도 변수를 선언하고 재사용할 수 있으며, JavaScript와 실시간으로 연동할 수 있어 동적인 스타일링이 훨씬 쉬워집니다. 이 글에서는 기본 문법부터 다크모드 구현, 실전 디자인 시스템 적용까지 단계적으로 살펴봅니다.

## CSS 커스텀 프로퍼티란?

CSS 커스텀 프로퍼티는 `--`로 시작하는 개발자가 직접 정의하는 CSS 속성입니다. `var()` 함수로 참조하며, 일반 CSS 속성처럼 상속과 캐스케이딩 규칙을 그대로 따릅니다.

```css
/* 선언 */
:root {
  --primary-color: #3b82f6;
  --font-size-base: 16px;
}

/* 사용 */
button {
  background-color: var(--primary-color);
  font-size: var(--font-size-base);
}
```

Sass 변수(`$primary-color`)와의 가장 큰 차이점은 **런타임에 동적으로 변경 가능**하다는 것입니다. Sass 변수는 컴파일 시점에 고정되지만, CSS 커스텀 프로퍼티는 브라우저에서 실행 중에도 JavaScript나 CSS 미디어 쿼리로 바꿀 수 있습니다.

## 스코프와 상속

### :root vs 로컬 스코프

`:root`에 선언하면 문서 전체에서 사용할 수 있는 전역 변수가 됩니다. 특정 컴포넌트 내부에서만 쓰이는 값은 해당 선택자에 선언하면 스코프를 제한할 수 있습니다.

```css
/* 전역 변수 */
:root {
  --spacing-unit: 8px;
}

/* 카드 컴포넌트 전용 변수 */
.card {
  --card-padding: calc(var(--spacing-unit) * 2);
  --card-radius: 12px;

  padding: var(--card-padding);
  border-radius: var(--card-radius);
}
```

자식 요소는 부모에서 선언한 커스텀 프로퍼티를 그대로 상속받습니다. 이 특성 덕분에 컴포넌트 단위 테마 설정이 매우 간결해집니다.

### 폴백(Fallback) 값

`var()` 함수의 두 번째 인자로 폴백 값을 지정할 수 있습니다. 변수가 정의되지 않은 환경에서 안전하게 동작합니다.

```css
.button {
  /* --brand-color가 없으면 #6366f1 사용 */
  background: var(--brand-color, #6366f1);
}
```

## 다크모드 구현하기

CSS 커스텀 프로퍼티의 가장 강력한 활용 사례는 다크모드입니다. 색상 값만 바꿔치기하면 전체 테마가 자동으로 업데이트됩니다.

```css
:root {
  --bg-primary: #ffffff;
  --text-primary: #111827;
  --surface: #f9fafb;
  --border: #e5e7eb;
}

@media (prefers-color-scheme: dark) {
  :root {
    --bg-primary: #111827;
    --text-primary: #f9fafb;
    --surface: #1f2937;
    --border: #374151;
  }
}

body {
  background-color: var(--bg-primary);
  color: var(--text-primary);
}
```

사용자가 토글 버튼으로 테마를 바꿀 수 있게 JavaScript로 클래스를 추가하는 방식도 많이 씁니다.

```css
[data-theme="dark"] {
  --bg-primary: #111827;
  --text-primary: #f9fafb;
}
```

```js
document.documentElement.setAttribute('data-theme', 'dark');
```

## JavaScript와 실시간 연동

CSS 커스텀 프로퍼티는 JavaScript에서 직접 읽고 쓸 수 있습니다.

```js
// 읽기
const primary = getComputedStyle(document.documentElement)
  .getPropertyValue('--primary-color').trim();

// 쓰기 (실시간 반영)
document.documentElement.style.setProperty('--primary-color', '#ef4444');
```

이 특성을 활용하면 사용자가 슬라이더를 움직일 때 `--font-size`가 즉시 반영되거나, 마우스 위치에 따라 그라디언트 방향이 바뀌는 인터랙션을 CSS만으로 구현할 수 있습니다.

## 실전 디자인 토큰 시스템 구축

대규모 프로젝트에서는 커스텀 프로퍼티를 디자인 토큰으로 활용해 일관된 시각 언어를 유지합니다.

```css
:root {
  /* 색상 팔레트 */
  --color-blue-500: #3b82f6;
  --color-blue-600: #2563eb;

  /* 의미 기반 토큰 */
  --color-action-primary: var(--color-blue-500);
  --color-action-primary-hover: var(--color-blue-600);

  /* 타이포그래피 */
  --text-xs: 0.75rem;
  --text-sm: 0.875rem;
  --text-base: 1rem;
  --text-lg: 1.125rem;

  /* 간격 */
  --space-1: 4px;
  --space-2: 8px;
  --space-4: 16px;
  --space-8: 32px;
}
```

원시 값(팔레트)과 의미 값(시멘틱 토큰)을 분리하면 나중에 브랜드 색상이 바뀌어도 팔레트 부분만 수정하면 됩니다.

## 주의할 점

- **CSS 커스텀 프로퍼티는 타입이 없습니다.** `calc()` 안에서 단위 없는 숫자를 쓸 때는 주의가 필요합니다.
- **IE11은 지원하지 않습니다.** 레거시 환경을 지원해야 한다면 폴백을 별도로 작성하세요.
- 너무 많은 전역 변수를 `:root`에 쌓으면 관리가 어려워집니다. 파일을 `tokens.css`, `theme.css` 등으로 분리하는 것을 권장합니다.

## 마치며

CSS 커스텀 프로퍼티는 전처리기 없이도 유지보수하기 좋은 스타일시트를 작성할 수 있게 해주는 강력한 도구입니다. 다크모드, 테마, 반응형 타이포그래피 같은 기능을 구현할 때 코드량을 대폭 줄여줍니다. 아직 `var()`를 자주 쓰지 않는다면, 오늘 당장 프로젝트의 색상·간격·폰트 크기를 커스텀 프로퍼티로 바꿔보세요.
