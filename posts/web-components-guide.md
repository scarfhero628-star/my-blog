---
title: 웹 컴포넌트 완전 가이드 — 프레임워크 없이 재사용 가능한 UI 컴포넌트 만들기
date: 2026-11-16
tags: [Web Components, 커스텀 엘리먼트, Shadow DOM, 웹 개발, 프론트엔드]
description: 브라우저 내장 Web Components API로 프레임워크에 의존하지 않는 재사용 가능한 UI 컴포넌트를 만드는 핵심 개념과 실전 활용법을 소개합니다.
---

React, Vue, Angular 같은 프레임워크 없이도 재사용 가능한 UI 컴포넌트를 만들 수 있다면 어떨까요? 바로 그것이 **웹 컴포넌트(Web Components)**가 가능하게 해주는 일입니다. 웹 컴포넌트는 브라우저가 기본으로 제공하는 세 가지 API — Custom Elements, Shadow DOM, HTML Templates — 를 조합해 완전히 캡슐화된 UI 블록을 만드는 표준 기술입니다.

## 웹 컴포넌트가 필요한 이유

현대 웹 개발에서 컴포넌트 기반 아키텍처는 필수입니다. 하지만 React나 Vue 없이는 컴포넌트를 만들 수 없다고 생각하는 개발자가 많습니다. 웹 컴포넌트는 그 고정관념을 깹니다.

### 주요 장점

- **프레임워크 독립성**: React, Vue, Angular, Svelte 어디서든 `<my-button>` 같은 커스텀 태그를 그대로 쓸 수 있습니다.
- **완전한 캡슐화**: Shadow DOM 덕분에 스타일과 로직이 외부에 영향을 주지 않고 격리됩니다.
- **브라우저 기본 지원**: 별도 번들 없이 모던 브라우저에서 바로 동작합니다.
- **장수 코드**: 프레임워크가 버전 업하거나 사라져도 웹 표준은 유지됩니다.

---

## 세 가지 핵심 API

### 1. Custom Elements — 나만의 HTML 태그 만들기

`customElements.define()`으로 완전히 새로운 HTML 엘리먼트를 등록합니다.

```js
class MyCard extends HTMLElement {
  connectedCallback() {
    this.innerHTML = `
      <div class="card">
        <h2>${this.getAttribute('title')}</h2>
        <p><slot></slot></p>
      </div>
    `;
  }
}

customElements.define('my-card', MyCard);
```

이제 HTML에서 아래처럼 쓸 수 있습니다.

```html
<my-card title="안녕하세요">
  웹 컴포넌트로 만든 카드입니다.
</my-card>
```

#### 라이프사이클 콜백

| 콜백 | 호출 시점 |
|------|-----------|
| `connectedCallback` | DOM에 추가될 때 |
| `disconnectedCallback` | DOM에서 제거될 때 |
| `attributeChangedCallback` | 속성 값이 변경될 때 |
| `adoptedCallback` | 다른 document로 이동될 때 |

속성 변화를 감지하려면 `observedAttributes` 정적 프로퍼티를 선언해야 합니다.

```js
class MyCounter extends HTMLElement {
  static get observedAttributes() {
    return ['count'];
  }

  attributeChangedCallback(name, oldValue, newValue) {
    if (name === 'count') {
      this.render();
    }
  }

  render() {
    const count = parseInt(this.getAttribute('count') || '0', 10);
    this.shadowRoot.querySelector('span').textContent = count;
  }
}
```

---

### 2. Shadow DOM — 완전한 스타일 캡슐화

Shadow DOM은 컴포넌트 내부 DOM 트리를 외부와 완전히 격리합니다. 외부 CSS가 내부에 침투하지 않고, 내부 스타일도 밖으로 새어나가지 않습니다.

```js
class MyButton extends HTMLElement {
  constructor() {
    super();
    // Shadow Root 생성
    const shadow = this.attachShadow({ mode: 'open' });

    shadow.innerHTML = `
      <style>
        button {
          background: #0070f3;
          color: white;
          border: none;
          border-radius: 6px;
          padding: 8px 16px;
          cursor: pointer;
          font-size: 14px;
        }
        button:hover {
          background: #005bb5;
        }
      </style>
      <button><slot></slot></button>
    `;
  }
}

customElements.define('my-button', MyButton);
```

이 버튼 컴포넌트의 `button { ... }` 스타일은 외부 페이지의 버튼에 절대 영향을 미치지 않습니다.

#### `mode: 'open'` vs `mode: 'closed'`

- `open`: `element.shadowRoot`로 외부에서 접근 가능
- `closed`: `element.shadowRoot`가 `null`을 반환해 외부 접근 차단

---

### 3. HTML Templates — 렌더링 전 마크업 선언

`<template>` 태그 안의 콘텐츠는 DOM에 추가되기 전까지 파싱되지만 렌더링되지 않습니다. 반복적으로 복제해 사용하기 적합합니다.

```html
<template id="user-card-template">
  <style>
    .user-card { border: 1px solid #ddd; padding: 16px; border-radius: 8px; }
    .avatar { width: 48px; height: 48px; border-radius: 50%; }
  </style>
  <div class="user-card">
    <img class="avatar" />
    <div class="info">
      <strong class="name"></strong>
      <small class="role"></small>
    </div>
  </div>
</template>
```

```js
class UserCard extends HTMLElement {
  connectedCallback() {
    const template = document.getElementById('user-card-template');
    const clone = template.content.cloneNode(true);

    clone.querySelector('.name').textContent = this.getAttribute('name');
    clone.querySelector('.role').textContent = this.getAttribute('role');
    clone.querySelector('.avatar').src = this.getAttribute('avatar');

    this.attachShadow({ mode: 'open' }).appendChild(clone);
  }
}

customElements.define('user-card', UserCard);
```

---

## 슬롯(Slot)으로 콘텐츠 주입하기

React의 `children`처럼, 웹 컴포넌트에서는 `<slot>`으로 외부에서 콘텐츠를 주입합니다.

```html
<!-- 컴포넌트 정의 내부 -->
<div class="dialog">
  <header><slot name="header">기본 제목</slot></header>
  <main><slot></slot></main>
  <footer><slot name="footer"></slot></footer>
</div>
```

```html
<!-- 사용 시 -->
<my-dialog>
  <span slot="header">경고</span>
  <p>저장하지 않은 변경사항이 있습니다.</p>
  <button slot="footer">확인</button>
</my-dialog>
```

이름 없는 `<slot>`은 기본 슬롯으로, `slot` 속성이 없는 모든 자식 노드를 담습니다.

---

## 실전 예제: 토글 스위치 컴포넌트

세 가지 API를 모두 조합한 완성도 높은 토글 컴포넌트를 만들어봅니다.

```js
class ToggleSwitch extends HTMLElement {
  static get observedAttributes() {
    return ['checked', 'disabled'];
  }

  constructor() {
    super();
    this.attachShadow({ mode: 'open' });
  }

  connectedCallback() {
    this.render();
    this.shadowRoot.querySelector('input').addEventListener('change', (e) => {
      if (this.hasAttribute('disabled')) return;
      this.toggleAttribute('checked', e.target.checked);
      this.dispatchEvent(new CustomEvent('change', {
        detail: { checked: e.target.checked },
        bubbles: true,
        composed: true, // Shadow DOM 경계 넘어 이벤트 전파
      }));
    });
  }

  attributeChangedCallback() {
    if (this.shadowRoot) this.render();
  }

  render() {
    const checked = this.hasAttribute('checked');
    const disabled = this.hasAttribute('disabled');

    this.shadowRoot.innerHTML = `
      <style>
        label { display: inline-flex; align-items: center; cursor: ${disabled ? 'not-allowed' : 'pointer'}; gap: 8px; }
        input { display: none; }
        .track {
          width: 44px; height: 24px;
          background: ${checked ? '#0070f3' : '#ccc'};
          border-radius: 12px;
          position: relative;
          transition: background 0.2s;
          opacity: ${disabled ? '0.5' : '1'};
        }
        .thumb {
          width: 18px; height: 18px;
          background: white;
          border-radius: 50%;
          position: absolute;
          top: 3px;
          left: ${checked ? '23px' : '3px'};
          transition: left 0.2s;
        }
      </style>
      <label>
        <input type="checkbox" ${checked ? 'checked' : ''} ${disabled ? 'disabled' : ''}>
        <span class="track"><span class="thumb"></span></span>
        <slot></slot>
      </label>
    `;
  }
}

customElements.define('toggle-switch', ToggleSwitch);
```

```html
<toggle-switch checked>다크 모드</toggle-switch>
<toggle-switch disabled>알림 (비활성)</toggle-switch>
```

---

## 프레임워크와 함께 사용하기

웹 컴포넌트의 가장 큰 강점 중 하나는 프레임워크와의 호환성입니다.

### React에서 사용

```jsx
function App() {
  const handleChange = (e) => {
    console.log('토글 상태:', e.detail.checked);
  };

  useEffect(() => {
    const el = document.querySelector('toggle-switch');
    el.addEventListener('change', handleChange);
    return () => el.removeEventListener('change', handleChange);
  }, []);

  return <toggle-switch ref={/* ref */}>알림</toggle-switch>;
}
```

React 19부터는 커스텀 이벤트와 프로퍼티 바인딩이 대폭 개선되어 웹 컴포넌트를 훨씬 자연스럽게 쓸 수 있습니다.

### Vue에서 사용

Vue는 웹 컴포넌트 친화적으로 설계되어 있어 별도 처리 없이 사용 가능합니다.

```vue
<template>
  <toggle-switch :checked="isDark" @change="onToggle">
    다크 모드
  </toggle-switch>
</template>
```

---

## 자주 발생하는 문제와 해결책

### 1. 초기화 타이밍 문제

`constructor`에서 속성을 읽으려 하면 아직 속성이 설정되지 않은 경우가 있습니다. `connectedCallback`이나 `attributeChangedCallback`에서 읽어야 합니다.

```js
// 잘못된 방법
constructor() {
  super();
  const title = this.getAttribute('title'); // 아직 null일 수 있음
}

// 올바른 방법
connectedCallback() {
  const title = this.getAttribute('title'); // 안전
}
```

### 2. `composed: true` 없는 이벤트

Shadow DOM 내부에서 발생한 이벤트는 기본적으로 경계를 넘지 못합니다. 외부로 전달하려면 `composed: true`를 설정해야 합니다.

```js
this.dispatchEvent(new CustomEvent('my-event', {
  bubbles: true,
  composed: true, // 필수
  detail: { ... }
}));
```

### 3. CSS 커스텀 프로퍼티로 외부 스타일링 허용

Shadow DOM이 외부 스타일을 차단하지만, CSS 커스텀 프로퍼티(변수)는 Shadow DOM 경계를 통과합니다. 이를 활용해 테마 커스터마이징 포인트를 열어줄 수 있습니다.

```css
/* 컴포넌트 내부 */
button {
  background: var(--my-button-bg, #0070f3);
  color: var(--my-button-color, white);
}
```

```css
/* 외부 페이지 */
my-button {
  --my-button-bg: #e74c3c;
  --my-button-color: #fff;
}
```

---

## 웹 컴포넌트 도입 시 고려할 점

웹 컴포넌트는 강력하지만 모든 상황에 최선이 아닐 수 있습니다.

**적합한 경우**:
- 여러 프레임워크에서 공유하는 디자인 시스템 컴포넌트
- CMS나 위젯처럼 외부 환경에 임베드되는 UI
- 프레임워크 없이 동작해야 하는 간단한 인터랙티브 요소

**신중히 검토할 경우**:
- 복잡한 상태 관리가 필요한 대규모 SPA (React/Vue 쪽이 생태계가 풍부)
- SSR(Server-Side Rendering) 환경 (웹 컴포넌트는 클라이언트 렌더링 위주)
- 팀 전체가 이미 특정 프레임워크에 익숙한 경우

---

## 마치며

웹 컴포넌트는 "프레임워크의 대안"이기보다 **프레임워크와 공존하는 표준 기반 도구**입니다. 특히 여러 프로젝트에서 공유해야 하는 디자인 시스템을 구축할 때, 웹 컴포넌트로 만든 컴포넌트는 어떤 프레임워크에서도 그대로 쓸 수 있다는 점이 압도적인 장점입니다. Custom Elements, Shadow DOM, HTML Templates — 이 세 가지 API를 익혀 두면 프레임워크 종속성에서 한 발짝 자유로워질 수 있습니다.
