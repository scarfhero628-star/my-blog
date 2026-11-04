---
title: CSS Container Queries 완전 가이드 — 컴포넌트 중심 반응형 디자인의 새로운 패러다임
date: 2026-11-04
tags: [CSS, Container Queries, 반응형 디자인, 웹 개발, 프론트엔드]
description: 뷰포트가 아닌 부모 컨테이너 크기에 반응하는 CSS Container Queries로 진정한 컴포넌트 중심 반응형 디자인을 구현하는 방법을 소개합니다.
---

## 왜 Container Queries가 필요한가?

지금까지 반응형 디자인은 미디어 쿼리(`@media`)에 의존했습니다. 뷰포트 너비를 기준으로 레이아웃을 조정하는 방식이죠. 그런데 컴포넌트 기반 개발이 대세가 된 지금, 미디어 쿼리만으로는 한계가 있습니다.

예를 들어 카드 컴포넌트를 생각해봅시다. 메인 페이지에서는 전체 너비 3칸 중 1칸을 차지하고, 사이드바에서는 전체 너비를 차지합니다. 뷰포트 기준 미디어 쿼리로는 카드가 실제로 얼마나 큰 공간에 렌더링되는지 알 수 없습니다. 바로 이 문제를 **CSS Container Queries**가 해결합니다.

---

## Container Queries 기본 사용법

### 1단계: 컨테이너 등록

```css
.card-wrapper {
  container-type: inline-size;
  container-name: card;
}
```

`container-type`에는 세 가지 값이 있습니다.

- `inline-size` — 인라인 축(보통 너비) 변화에만 반응. 가장 많이 쓰임
- `size` — 너비와 높이 모두 반응
- `normal` — 쿼리 불가, 기본값

`container-name`은 선택 사항이지만 중첩된 컨테이너가 있을 때 명시적으로 지정하면 혼란을 방지할 수 있습니다.

### 2단계: 컨테이너 쿼리 작성

```css
/* 컨테이너 너비가 400px 미만이면 세로 배치 */
@container (max-width: 400px) {
  .card {
    flex-direction: column;
  }
}

/* 특정 컨테이너를 대상으로 지정 */
@container card (min-width: 600px) {
  .card__title {
    font-size: 1.5rem;
  }
}
```

미디어 쿼리와 문법이 거의 같아서 기존 개발자라면 쉽게 익힐 수 있습니다.

---

## 실전 예제: 재사용 가능한 카드 컴포넌트

```html
<div class="layout-grid">
  <article class="card-wrapper">
    <div class="card">
      <img class="card__image" src="thumbnail.jpg" alt="썸네일" />
      <div class="card__body">
        <h2 class="card__title">포스트 제목</h2>
        <p class="card__desc">짧은 설명 텍스트가 들어갑니다.</p>
      </div>
    </div>
  </article>
</div>
```

```css
.card-wrapper {
  container-type: inline-size;
}

.card {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.card__image {
  width: 100%;
  aspect-ratio: 16 / 9;
  object-fit: cover;
  border-radius: 8px;
}

/* 컨테이너가 충분히 넓으면 가로 배치 */
@container (min-width: 480px) {
  .card {
    flex-direction: row;
    align-items: flex-start;
  }

  .card__image {
    width: 200px;
    aspect-ratio: 1;
    flex-shrink: 0;
  }
}
```

이 카드 컴포넌트는 어디에 배치하든 **자신이 가진 공간에 맞게** 자동으로 레이아웃을 전환합니다. 뷰포트 크기를 전혀 신경 쓰지 않아도 됩니다.

---

## Container Query Units

Container Queries와 함께 새로운 단위도 도입됐습니다.

| 단위 | 의미 |
|------|------|
| `cqi` | 컨테이너 인라인 크기의 1% |
| `cqb` | 컨테이너 블록 크기의 1% |
| `cqw` | 컨테이너 너비의 1% |
| `cqh` | 컨테이너 높이의 1% |
| `cqmin` | cqw와 cqh 중 더 작은 값 |
| `cqmax` | cqw와 cqh 중 더 큰 값 |

`vw`/`vh`의 컨테이너 버전이라고 이해하면 쉽습니다. `clamp()`와 함께 쓰면 컨테이너 크기에 비례하는 유동적인 타이포그래피를 만들 수 있습니다.

```css
/* 컨테이너 너비에 비례한 폰트 크기 */
.card__title {
  font-size: clamp(1rem, 5cqi, 2rem);
}
```

---

## 미디어 쿼리와 함께 쓰기

Container Queries와 미디어 쿼리는 경쟁 관계가 아닙니다. 각자의 역할이 다릅니다.

- **미디어 쿼리** — 페이지 전체 레이아웃, 헤더/푸터 구조, 사이드바 존재 여부
- **Container Queries** — 개별 컴포넌트 내부 레이아웃

```css
/* 뷰포트 기준으로 전체 그리드 구성 */
@media (min-width: 768px) {
  .layout-grid {
    grid-template-columns: 1fr 300px;
  }
}

/* 각 카드는 자신의 공간에 맞게 반응 */
@container (min-width: 400px) {
  .card {
    flex-direction: row;
  }
}
```

이렇게 역할을 명확히 분리하면 레이아웃 코드와 컴포넌트 코드가 서로 간섭하지 않아 유지보수가 훨씬 쉬워집니다.

---

## 스타일 쿼리(Style Queries)

Container Queries는 크기뿐 아니라 CSS 커스텀 프로퍼티 값에도 반응할 수 있습니다. 아직 실험적 기능이지만 알아두면 유용합니다.

```css
/* 부모 컨테이너의 커스텀 프로퍼티 값에 따라 스타일 변경 */
@container style(--theme: dark) {
  .card {
    background: #1a1a2e;
    color: #eee;
  }
}
```

JavaScript 없이도 컨테이너의 "상태"에 따라 자식 컴포넌트 스타일을 바꿀 수 있어 흥미로운 패턴을 만들 수 있습니다.

---

## 브라우저 지원 현황과 폴백 처리

2023년부터 Chrome, Firefox, Safari 등 모든 주요 브라우저에서 Container Queries를 지원합니다. 현재 글로벌 지원율은 90% 이상으로 프로덕션 환경에서 안심하고 사용할 수 있습니다.

구형 브라우저를 위한 폴백이 필요하다면 `@supports`와 함께 사용하세요.

```css
/* 폴백: 기본 세로 배치 */
.card {
  display: flex;
  flex-direction: column;
}

/* Container Queries 지원 환경에서만 적용 */
@supports (container-type: inline-size) {
  .card-wrapper {
    container-type: inline-size;
  }

  @container (min-width: 480px) {
    .card {
      flex-direction: row;
    }
  }
}
```

---

## 정리

Container Queries는 "컴포넌트가 어디에 배치되든 올바르게 보이게 하려면 어떻게 해야 하는가?"라는 오래된 질문에 CSS가 내놓은 답입니다.

### 핵심 포인트 요약

- `container-type: inline-size`로 컨테이너를 등록한다
- `@container (조건)` 으로 컨테이너 크기에 반응하는 스타일을 작성한다
- `cqi`, `cqw` 등 새 단위를 활용해 컨테이너 비례 크기를 설정한다
- 미디어 쿼리는 전체 레이아웃, Container Queries는 컴포넌트에 적용해 역할을 분리한다

디자인 시스템을 구축하거나 재사용 가능한 컴포넌트를 만들 때 Container Queries는 특히 위력을 발휘합니다. 아직 프로젝트에 도입하지 않았다면, 다음 카드나 위젯 컴포넌트 작업에서 한번 적용해보세요. 뷰포트 걱정 없이 컴포넌트 자체에만 집중할 수 있는 개발 경험의 차이를 바로 느낄 수 있을 것입니다.
