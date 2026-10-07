---
title: CSS Flexbox 완전 가이드 — 레이아웃의 기본기를 제대로 익히자
date: 2026-10-07
tags: [CSS, Flexbox, 웹 개발, 레이아웃, 프론트엔드]
description: CSS Flexbox의 핵심 개념과 자주 쓰이는 실전 패턴을 정리해 복잡한 레이아웃도 간결하게 작성하는 방법을 소개합니다.
---

CSS 레이아웃을 다루다 보면 "이걸 어떻게 가운데 정렬하지?" 또는 "왜 자식 요소가 삐져나오지?"라는 상황을 자주 겪게 됩니다. Flexbox는 이런 레이아웃 문제를 직관적으로 해결해 주는 강력한 CSS 도구입니다. CSS Grid가 2차원 레이아웃에 특화되어 있다면, Flexbox는 **단일 축(행 또는 열)** 방향의 배치에 최적화되어 있습니다.

## Flexbox의 두 주인공: 컨테이너와 아이템

Flexbox는 **플렉스 컨테이너(Flex Container)** 와 **플렉스 아이템(Flex Item)** 이라는 두 가지 개념을 중심으로 돌아갑니다.

```css
/* 컨테이너에 display: flex 선언 */
.container {
  display: flex;
}

/* 컨테이너의 직계 자식이 자동으로 플렉스 아이템이 됨 */
.item {
  /* 별도 선언 없어도 플렉스 아이템으로 동작 */
}
```

`display: flex`를 선언하는 순간, 컨테이너 안의 모든 직계 자식 요소는 플렉스 아이템이 됩니다. 기본값으로 아이템들은 가로(행) 방향으로 나란히 배치됩니다.

---

## 핵심 컨테이너 속성

### flex-direction — 정렬 방향 설정

```css
.container {
  display: flex;
  flex-direction: row;         /* 기본값: 왼쪽 → 오른쪽 */
  /* flex-direction: row-reverse;  오른쪽 → 왼쪽 */
  /* flex-direction: column;       위 → 아래 */
  /* flex-direction: column-reverse; 아래 → 위 */
}
```

### justify-content — 주축(Main Axis) 정렬

주축은 `flex-direction`이 결정합니다. `row`면 가로가 주축, `column`이면 세로가 주축입니다.

```css
.container {
  display: flex;
  justify-content: flex-start;    /* 기본값 */
  justify-content: flex-end;      /* 끝 정렬 */
  justify-content: center;        /* 가운데 정렬 */
  justify-content: space-between; /* 양 끝에 붙고 사이 균등 분배 */
  justify-content: space-around;  /* 각 아이템 양쪽에 균등 여백 */
  justify-content: space-evenly;  /* 모든 간격 완전 균등 */
}
```

### align-items — 교차축(Cross Axis) 정렬

교차축은 주축의 수직 방향입니다.

```css
.container {
  display: flex;
  align-items: stretch;     /* 기본값: 교차축 방향으로 늘어남 */
  align-items: flex-start;  /* 위쪽(또는 왼쪽) 정렬 */
  align-items: flex-end;    /* 아래쪽(또는 오른쪽) 정렬 */
  align-items: center;      /* 교차축 가운데 정렬 */
  align-items: baseline;    /* 텍스트 기준선에 맞춤 */
}
```

### flex-wrap — 줄 바꿈 설정

```css
.container {
  display: flex;
  flex-wrap: nowrap;  /* 기본값: 한 줄에 모두 배치 (넘쳐도 줄 바꿈 안 함) */
  flex-wrap: wrap;    /* 넘치면 다음 줄로 */
}
```

---

## 핵심 아이템 속성

### flex — 크기 비율 지정

`flex` 속성은 `flex-grow`, `flex-shrink`, `flex-basis`의 단축 속성입니다.

```css
.item {
  flex: 1;        /* flex-grow: 1, flex-shrink: 1, flex-basis: 0% */
  flex: 0 1 auto; /* 기본값 */
  flex: 2;        /* 남은 공간을 다른 아이템보다 2배 더 차지 */
}

/* 실전: 세 아이템이 균등하게 영역을 나눠 갖기 */
.sidebar { flex: 1; }
.main    { flex: 2; }
```

### align-self — 개별 아이템 교차축 정렬

컨테이너의 `align-items`를 무시하고 특정 아이템만 따로 정렬할 수 있습니다.

```css
.special-item {
  align-self: flex-end; /* 이 아이템만 아래쪽 정렬 */
}
```

### order — 시각적 순서 변경

HTML 구조를 바꾸지 않고도 아이템의 순서를 바꿀 수 있습니다.

```css
.item-a { order: 2; }
.item-b { order: 1; } /* item-b가 item-a보다 먼저 표시됨 */
```

---

## 실전 패턴 3가지

### 1. 완전한 가운데 정렬 (수직 + 수평)

```css
.center-container {
  display: flex;
  justify-content: center;
  align-items: center;
  height: 100vh; /* 화면 전체 높이 */
}
```

### 2. 내비게이션 바 — 로고는 왼쪽, 메뉴는 오른쪽

```css
.navbar {
  display: flex;
  align-items: center;
  justify-content: space-between;
}

/* 또는 margin-left: auto 활용 */
.nav-menu {
  margin-left: auto;
}
```

### 3. 카드 그리드 (반응형)

```css
.card-list {
  display: flex;
  flex-wrap: wrap;
  gap: 16px;
}

.card {
  flex: 1 1 280px; /* 최소 280px, 남은 공간 균등 분배 */
}
```

---

## Flexbox vs Grid — 언제 무엇을 쓸까?

| 상황 | 추천 |
|------|------|
| 단일 행/열 배치 | Flexbox |
| 2차원 격자 레이아웃 | Grid |
| 내비게이션, 버튼 그룹 | Flexbox |
| 전체 페이지 레이아웃 | Grid |
| 아이템 크기가 내용에 따라 유동적 | Flexbox |

두 기술은 경쟁 관계가 아닙니다. **Flexbox로 컴포넌트 내부를 정렬하고, Grid로 페이지 전체 구조를 잡는** 방식이 실무에서 가장 많이 쓰입니다.

---

## 정리

Flexbox는 배우는 데 하루도 안 걸리지만, 레이아웃 작업의 80%를 해결해 줄 만큼 강력합니다. 핵심은 **주축과 교차축의 개념을 명확히 이해하는 것**입니다. `justify-content`는 주축을, `align-items`는 교차축을 제어한다는 것만 기억해도 대부분의 정렬 문제를 해결할 수 있습니다.

오늘 배운 속성들을 직접 [CSS Flexbox Froggy](https://flexboxfroggy.com/) 게임으로 연습해 보세요. 개구리를 정렬하다 보면 자연스럽게 손에 익습니다.
