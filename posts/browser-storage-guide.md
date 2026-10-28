---
title: 브라우저 스토리지 완전 가이드 — localStorage, sessionStorage, IndexedDB, Cookie 비교와 실전 활용법
date: 2026-10-28
tags: [브라우저 스토리지, localStorage, Cookie, IndexedDB, 웹 개발]
description: 브라우저가 제공하는 네 가지 저장소의 차이점과 상황에 맞는 선택 기준을 실전 예제와 함께 정리합니다.
---

웹 애플리케이션을 개발하다 보면 클라이언트 측에 데이터를 저장해야 할 상황이 자주 생깁니다. 사용자 설정, 인증 토큰, 오프라인 캐시 등 저장 목적은 다양하지만, 도구를 잘못 고르면 보안 허점이 생기거나 UX가 나빠집니다. 이 글에서는 브라우저가 제공하는 네 가지 저장소를 비교하고, 각각의 적절한 사용 시나리오를 실전 예제와 함께 살펴봅니다.

---

## 브라우저 스토리지 한눈에 비교

| 저장소 | 용량 | 만료 | 서버 전송 | 동기 여부 |
|---|---|---|---|---|
| Cookie | ~4 KB | 설정 가능 | 자동 전송 | 동기 |
| localStorage | ~5–10 MB | 영구 | 안 함 | 동기 |
| sessionStorage | ~5–10 MB | 탭 닫힘 시 | 안 함 | 동기 |
| IndexedDB | 수백 MB+ | 영구 | 안 함 | 비동기 |

---

## 1. Cookie

쿠키는 가장 오래된 브라우저 저장소로, HTTP 요청마다 서버로 자동 전송된다는 점이 핵심입니다.

### 언제 쓰나요?

- 세션 ID, 인증 토큰 (HttpOnly 쿠키로 XSS 방어)
- 서버가 읽어야 하는 사용자 설정 (언어, 지역 등)
- CSRF 방어용 토큰

### 실전 예제

```javascript
// 쿠키 설정 (JavaScript)
document.cookie = "theme=dark; path=/; max-age=2592000; SameSite=Lax";

// 쿠키 읽기
function getCookie(name) {
  return document.cookie
    .split("; ")
    .find(row => row.startsWith(name + "="))
    ?.split("=")[1];
}

// 서버에서 안전하게 설정 (Node.js/Express)
res.cookie("sessionId", token, {
  httpOnly: true,   // JS에서 접근 불가 → XSS 방어
  secure: true,     // HTTPS에서만 전송
  sameSite: "Strict",
  maxAge: 7 * 24 * 60 * 60 * 1000, // 7일
});
```

> **주의:** 인증 토큰을 JavaScript에서 읽을 수 있는 쿠키에 저장하면 XSS 공격에 취약합니다. 반드시 `HttpOnly` 플래그를 사용하세요.

---

## 2. localStorage

페이지를 닫고 다시 열어도 데이터가 남아 있는 영구 저장소입니다. 같은 도메인의 모든 탭이 공유합니다.

### 언제 쓰나요?

- 다크모드, 언어, 폰트 크기 등 사용자 UI 설정
- 장바구니 임시 저장
- 마지막 방문 페이지 기록

### 실전 예제

```javascript
// 저장 (JSON 직렬화 필수)
localStorage.setItem("userPrefs", JSON.stringify({
  theme: "dark",
  fontSize: 16,
}));

// 읽기 (파싱 실패 방어)
function getPrefs() {
  try {
    return JSON.parse(localStorage.getItem("userPrefs")) ?? {};
  } catch {
    return {};
  }
}

// 삭제
localStorage.removeItem("userPrefs");
```

> **주의:** `localStorage`는 동기 API라 대용량 데이터 처리 시 메인 스레드를 블로킹합니다. 수 KB 이상의 데이터는 IndexedDB를 고려하세요.

---

## 3. sessionStorage

`localStorage`와 API가 동일하지만, **탭을 닫으면 데이터가 사라집니다.** 같은 도메인이라도 탭끼리 공유되지 않습니다.

### 언제 쓰나요?

- 다단계 폼의 임시 입력값 저장
- 탭 단위 상태 관리 (로그인 세션 분리 등)
- 민감하지 않은 일회성 필터·정렬 조건

### 실전 예제

```javascript
// 다단계 폼 1단계에서 저장
sessionStorage.setItem("signupStep1", JSON.stringify({
  name: "홍길동",
  email: "hong@example.com",
}));

// 2단계에서 불러오기
const step1 = JSON.parse(sessionStorage.getItem("signupStep1") ?? "{}");

// 완료 후 정리
sessionStorage.removeItem("signupStep1");
```

---

## 4. IndexedDB

브라우저 내장 NoSQL 데이터베이스입니다. 비동기 API로 대용량 구조화 데이터를 처리할 수 있습니다.

### 언제 쓰나요?

- PWA 오프라인 캐시
- 대량 이미지·파일 바이너리 저장
- 복잡한 쿼리가 필요한 클라이언트 데이터

### 실전 예제 (idb 라이브러리 활용)

```javascript
import { openDB } from "idb";

const db = await openDB("my-app", 1, {
  upgrade(db) {
    db.createObjectStore("articles", { keyPath: "id" });
  },
});

// 저장
await db.put("articles", { id: 1, title: "오프라인 글", body: "..." });

// 읽기
const article = await db.get("articles", 1);

// 전체 조회
const all = await db.getAll("articles");
```

> 브라우저 네이티브 IndexedDB API는 콜백 기반이라 복잡합니다. [idb](https://github.com/jakearchibald/idb) 같은 래퍼 라이브러리를 사용하면 Promise 기반으로 간결하게 쓸 수 있습니다.

---

## 어떤 저장소를 선택해야 할까?

```
인증 토큰 → HttpOnly Cookie
사용자 UI 설정 (영구) → localStorage
폼 임시 저장 (탭 단위) → sessionStorage
오프라인 캐시 / 대용량 데이터 → IndexedDB
```

### 자주 저지르는 실수

1. **localStorage에 토큰 저장** — XSS에 노출됩니다. HttpOnly Cookie를 사용하세요.
2. **쿠키에 대용량 데이터** — 매 요청마다 전송되어 대역폭을 낭비합니다.
3. **IndexedDB를 동기처럼 사용** — `await` 없이 값을 읽으면 `undefined`가 반환됩니다.
4. **try-catch 생략** — 시크릿 모드나 스토리지 용량 초과 시 예외가 발생합니다. 항상 방어 코드를 넣으세요.

---

## 마치며

브라우저 스토리지는 각자의 설계 목적이 뚜렷합니다. 용량·만료·서버 전송 여부 세 가지 축으로 판단하면 대부분 올바른 선택을 할 수 있습니다. 특히 인증 관련 데이터는 보안을 최우선으로 고려해 `HttpOnly Cookie`를 기본으로 삼고, 나머지 경우에는 데이터의 수명과 크기에 따라 `localStorage` → `IndexedDB` 순으로 확장하는 전략을 권장합니다.
