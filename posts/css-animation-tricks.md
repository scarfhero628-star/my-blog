---
title: CSS 애니메이션과 트랜지션 마스터하기 — JavaScript 없이 부드러운 인터랙션 구현
date: 2026-09-15
tags: [CSS, 애니메이션, 트랜지션, 웹 개발, 인터랙션]
description: CSS 애니메이션과 트랜지션을 활용해 JavaScript 없이 부드럽고 성능 좋은 인터랙션을 구현하는 실전 기법을 소개합니다.
---

## 왜 CSS 애니메이션인가?

많은 개발자가 버튼 호버 효과나 모달 등장 같은 UI 인터랙션을 JavaScript로 처리하려 합니다. 하지만 CSS만으로도 대부분의 UI 애니메이션을 구현할 수 있고, 성능 면에서도 훨씬 유리합니다. CSS 트랜지션과 애니메이션은 GPU 가속을 활용하기 때문에 메인 스레드를 차지하지 않아 스크롤이나 입력 처리를 방해하지 않습니다.

이 글에서는 실무에서 바로 활용할 수 있는 CSS 애니메이션 기법 7가지를 정리합니다.

---

## 1. `transition`의 기본 — 부드러운 상태 변환

`transition`은 요소의 CSS 속성값이 변할 때 즉시 바뀌는 대신 부드럽게 전환되도록 합니다.

```css
.button {
  background-color: #3b82f6;
  transform: scale(1);
  transition: background-color 0.2s ease, transform 0.15s ease;
}

.button:hover {
  background-color: #1d4ed8;
  transform: scale(1.05);
}
```

**핵심 팁:** `all`로 모든 속성에 트랜지션을 걸면 예상치 못한 속성(예: `height`, `display`)까지 적용돼 버그가 생길 수 있습니다. 항상 **특정 속성**만 명시하세요.

---

## 2. `transform`으로 레이아웃 변경 없이 움직이기

요소를 움직일 때 `top`, `left`, `margin` 대신 `transform: translate()`를 사용하면 레이아웃 재계산(reflow)을 피할 수 있습니다.

```css
/* 나쁜 예 — reflow 발생 */
.card:hover {
  margin-top: -4px;
}

/* 좋은 예 — GPU 가속 */
.card {
  transition: transform 0.2s ease;
}
.card:hover {
  transform: translateY(-4px);
}
```

`transform`과 `opacity`는 브라우저가 합성(compositing) 레이어에서 처리하기 때문에 특히 성능이 뛰어납니다.

---

## 3. `@keyframes`로 반복 애니메이션 만들기

`transition`이 상태 전환에 쓰인다면, `@keyframes`는 독립적으로 반복되는 애니메이션에 적합합니다.

```css
@keyframes pulse {
  0%, 100% {
    transform: scale(1);
    opacity: 1;
  }
  50% {
    transform: scale(1.08);
    opacity: 0.8;
  }
}

.badge--live {
  animation: pulse 1.8s ease-in-out infinite;
}
```

`animation` 단축 속성 순서: `이름 지속시간 타이밍함수 지연 반복횟수 방향 fill-mode`.

---

## 4. 등장·퇴장 애니메이션 — `@keyframes` + 클래스 전환

모달이나 토스트 같은 요소를 부드럽게 표시하려면 `@keyframes`와 클래스를 조합합니다.

```css
@keyframes fadeInUp {
  from {
    opacity: 0;
    transform: translateY(16px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.modal {
  animation: fadeInUp 0.25s ease forwards;
}
```

퇴장 애니메이션은 `animation-direction: reverse`를 활용하거나 별도 `fadeOutDown` 키프레임을 추가하면 됩니다.

---

## 5. CSS 변수로 애니메이션 재사용하기

같은 애니메이션을 여러 속도로 적용할 때 CSS 커스텀 프로퍼티(변수)를 활용하면 코드가 깔끔해집니다.

```css
:root {
  --duration-fast: 0.15s;
  --duration-normal: 0.3s;
  --duration-slow: 0.6s;
  --ease-out: cubic-bezier(0.16, 1, 0.3, 1);
}

.dropdown {
  transition: opacity var(--duration-normal) var(--ease-out),
              transform var(--duration-normal) var(--ease-out);
}

.tooltip {
  transition: opacity var(--duration-fast) var(--ease-out);
}
```

전체 사이트의 애니메이션 속도를 한 곳에서 조정할 수 있어 디자인 시스템 구축에 매우 유용합니다.

---

## 6. `prefers-reduced-motion` — 접근성 고려

일부 사용자는 시스템 설정에서 애니메이션 축소를 선택합니다. 이를 무시하면 어지럼증이나 집중 장애를 유발할 수 있습니다.

```css
@keyframes spin {
  to { transform: rotate(360deg); }
}

.loader {
  animation: spin 1s linear infinite;
}

@media (prefers-reduced-motion: reduce) {
  .loader {
    animation: none;
    /* 애니메이션 대신 정적 표시 */
    opacity: 0.6;
  }
}
```

또는 전역으로 간단히 처리할 수 있습니다.

```css
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration: 0.01ms !important;
    transition-duration: 0.01ms !important;
  }
}
```

---

## 7. `will-change`로 성능 최적화 — 단, 신중하게

애니메이션이 시작되기 전에 브라우저에게 미리 합성 레이어를 만들도록 힌트를 줄 수 있습니다.

```css
.animated-card {
  will-change: transform, opacity;
}
```

**주의:** `will-change`는 GPU 메모리를 추가로 사용합니다. 모든 요소에 남발하면 오히려 성능이 저하됩니다. **애니메이션 성능 문제가 실제로 측정된 경우에만** 적용하세요. 사용 후에는 JavaScript로 제거하는 것이 좋습니다.

```js
element.addEventListener('animationend', () => {
  element.style.willChange = 'auto';
});
```

---

## 정리 — CSS 애니메이션 체크리스트

| 상황 | 추천 방법 |
|---|---|
| 호버, 포커스 상태 전환 | `transition` |
| 반복되는 자동 애니메이션 | `@keyframes` + `animation` |
| 요소 이동 | `transform: translate()` |
| 등장·퇴장 효과 | `@keyframes` + 클래스 토글 |
| 접근성 대응 | `prefers-reduced-motion` 미디어 쿼리 |
| 성능 문제 발생 시 | `will-change` (측정 후 선택적으로) |

CSS 애니메이션은 배우기는 쉽지만 잘 쓰기는 어렵습니다. 처음에는 단순한 `transition`부터 시작해 점점 복잡한 `@keyframes`로 나아가는 것이 좋습니다. 무엇보다 **성능과 접근성**을 항상 염두에 두면서 애니메이션을 만들어 보세요.
