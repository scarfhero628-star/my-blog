---
title: 웹 보안 기초 — XSS·CSRF·SQL 인젝션을 막는 개발자 필수 방어 전략
date: 2026-09-28
tags: [웹 보안, XSS, CSRF, SQL 인젝션, 보안]
description: 실무에서 가장 자주 마주치는 세 가지 웹 공격 유형과 코드 레벨에서 바로 적용할 수 있는 방어 기법을 소개합니다.
---

보안은 "나중에 챙기면 되는 것"이 아닙니다. 한 번의 취약점이 사용자 데이터 유출, 서비스 중단, 법적 책임으로 이어질 수 있습니다. 특히 프론트엔드와 백엔드를 모두 다루는 풀스택 개발자라면 공격자의 시각으로 코드를 바라보는 습관이 필요합니다.

이번 포스트에서는 OWASP(Open Web Application Security Project)가 꾸준히 상위권으로 꼽는 세 가지 취약점 — **XSS**, **CSRF**, **SQL 인젝션** — 의 작동 원리와 코드 레벨 방어 기법을 정리합니다.

---

## 1. XSS (Cross-Site Scripting)

### 어떻게 동작하나?

XSS는 공격자가 악의적인 스크립트를 웹 페이지에 삽입해 다른 사용자의 브라우저에서 실행시키는 공격입니다. 가장 흔한 시나리오는 사용자 입력을 그대로 HTML에 출력하는 경우입니다.

```html
<!-- 위험한 코드: 사용자 입력을 그대로 렌더링 -->
<div id="comment"></div>
<script>
  document.getElementById('comment').innerHTML = userInput;
  // userInput = "<script>document.cookie를 외부로 전송...</script>"
</script>
```

### 방어 방법

**① 출력 시 이스케이프(Escape) 처리**

HTML에 동적 데이터를 삽입할 때는 반드시 특수 문자를 이스케이프합니다.

```javascript
function escapeHtml(str) {
  return str
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;')
    .replace(/'/g, '&#039;');
}

// 안전한 사용
element.textContent = userInput;      // textContent는 자동 이스케이프
element.innerHTML = escapeHtml(userInput); // innerHTML 사용 시 수동 처리
```

**② Content Security Policy (CSP) 헤더 설정**

서버에서 CSP 헤더를 설정하면 허가되지 않은 스크립트 실행을 브라우저 레벨에서 차단할 수 있습니다.

```http
Content-Security-Policy: default-src 'self'; script-src 'self' https://trusted-cdn.com
```

**③ DOMPurify 같은 라이브러리 활용**

사용자가 입력한 HTML(예: 에디터)을 그대로 렌더링해야 한다면 DOMPurify로 정제합니다.

```javascript
import DOMPurify from 'dompurify';
element.innerHTML = DOMPurify.sanitize(userInput);
```

---

## 2. CSRF (Cross-Site Request Forgery)

### 어떻게 동작하나?

CSRF는 이미 로그인된 사용자가 자신도 모르게 악의적인 요청을 보내도록 유도하는 공격입니다. 예를 들어 피싱 페이지에 다음 코드가 숨겨져 있다면, 사용자가 접속하는 순간 본인 계정에서 송금이 실행될 수 있습니다.

```html
<!-- 피싱 페이지에 숨겨진 자동 제출 폼 -->
<form action="https://bank.example.com/transfer" method="POST" id="evil">
  <input name="amount" value="1000000">
  <input name="to" value="attacker-account">
</form>
<script>document.getElementById('evil').submit();</script>
```

### 방어 방법

**① CSRF 토큰 사용**

서버가 세션마다 고유한 토큰을 발급하고, 상태를 변경하는 모든 요청에 토큰 검증을 요구합니다.

```javascript
// Express + csurf 예시
const csrf = require('csurf');
app.use(csrf({ cookie: true }));

app.get('/form', (req, res) => {
  res.render('form', { csrfToken: req.csrfToken() });
});

// 폼에서 토큰 전송
// <input type="hidden" name="_csrf" value="{{ csrfToken }}">
```

**② SameSite 쿠키 속성 설정**

쿠키에 `SameSite=Strict` 또는 `SameSite=Lax`를 설정하면 다른 사이트에서 시작된 요청에 쿠키가 전송되지 않습니다.

```http
Set-Cookie: sessionId=abc123; SameSite=Strict; Secure; HttpOnly
```

**③ 상태 변경 요청은 POST/PUT/DELETE만 허용**

GET 요청으로 데이터를 변경하는 설계를 피하세요. REST 원칙을 따르면 CSRF 공격 표면이 크게 줄어듭니다.

---

## 3. SQL 인젝션

### 어떻게 동작하나?

사용자 입력을 SQL 쿼리에 직접 삽입하면 공격자가 쿼리 구조 자체를 변조할 수 있습니다.

```javascript
// 위험한 코드
const query = `SELECT * FROM users WHERE id = '${userId}'`;
// userId = "1' OR '1'='1"
// 결과 쿼리: SELECT * FROM users WHERE id = '1' OR '1'='1'
// → 모든 사용자 데이터가 반환됨
```

더 심각한 경우 데이터 삭제나 테이블 구조 노출도 가능합니다.

### 방어 방법

**① Prepared Statement (파라미터화 쿼리) 사용**

가장 확실한 방어책입니다. 사용자 입력을 SQL 로직과 분리합니다.

```javascript
// 안전한 코드 — mysql2 예시
const [rows] = await connection.execute(
  'SELECT * FROM users WHERE id = ?',
  [userId]  // 값은 별도 파라미터로 전달
);

// Prisma, TypeORM 같은 ORM도 기본적으로 파라미터화 쿼리를 사용
const user = await prisma.user.findUnique({ where: { id: userId } });
```

**② 입력 유효성 검증**

예상 형식에 맞지 않는 입력은 아예 거부합니다.

```javascript
// userId는 숫자여야 한다
if (!/^\d+$/.test(userId)) {
  return res.status(400).json({ error: 'Invalid user ID' });
}
```

**③ 최소 권한 원칙 적용**

애플리케이션에서 사용하는 DB 계정에는 필요한 최소 권한만 부여합니다. `DROP TABLE` 같은 위험한 명령을 실행할 수 없는 계정을 사용하면 피해를 최소화할 수 있습니다.

---

## 4. 그 외 꼭 챙겨야 할 보안 기본기

### HTTPS 강제 사용

```javascript
// Express에서 HTTP → HTTPS 리다이렉트
app.use((req, res, next) => {
  if (req.headers['x-forwarded-proto'] !== 'https') {
    return res.redirect(`https://${req.headers.host}${req.url}`);
  }
  next();
});
```

### 비밀번호는 반드시 해싱 저장

```javascript
const bcrypt = require('bcrypt');

// 저장 시
const hashedPassword = await bcrypt.hash(plainPassword, 12);

// 검증 시
const isMatch = await bcrypt.compare(plainPassword, hashedPassword);
```

### 민감 정보는 환경 변수로 관리

API 키, DB 비밀번호, JWT 시크릿은 절대 코드에 하드코딩하지 말고 `.env` 파일과 환경 변수로 관리하세요. `.env` 파일은 반드시 `.gitignore`에 추가해야 합니다.

```bash
# .env
DATABASE_URL=postgres://user:password@localhost/mydb
JWT_SECRET=super-secret-key

# .gitignore
.env
.env.local
```

---

## 마치며

웹 보안은 한 번 적용하고 끝나는 체크리스트가 아니라 지속적인 관심이 필요한 분야입니다. 새로운 라이브러리를 도입할 때마다 `npm audit`을 실행하고, OWASP Top 10을 주기적으로 살펴보는 습관을 들여보세요.

가장 중요한 원칙은 단순합니다. **사용자 입력은 항상 신뢰하지 말고 검증하라.** 이 하나의 원칙을 습관으로 만들면 대부분의 공격을 예방할 수 있습니다.
