---
title: JWT와 OAuth로 안전한 인증 구현하기 — 토큰 기반 인증의 원리와 실전 패턴
date: 2026-10-10
tags: [JWT, OAuth, 인증, 보안, 웹 개발]
description: JWT의 구조와 OAuth 2.0 흐름을 이해하고, 실무에서 바로 적용할 수 있는 안전한 토큰 기반 인증 패턴을 소개합니다.
---

로그인 기능을 구현할 때 가장 먼저 맞닥뜨리는 질문이 있습니다. "세션을 쓸까, 토큰을 쓸까?" 현대 웹 애플리케이션, 특히 SPA(Single Page Application)나 모바일 앱과 연동되는 API 서버에서는 **JWT(JSON Web Token)** 와 **OAuth 2.0** 이 사실상 표준으로 자리잡았습니다. 이 글에서는 두 기술의 핵심 개념과 실전 구현 패턴을 단계별로 정리합니다.

---

## JWT란 무엇인가

JWT는 당사자 간에 정보를 JSON 형식으로 안전하게 전달하기 위한 **자기 포함형(self-contained) 토큰** 입니다. 서버가 별도로 세션 저장소를 조회하지 않아도 토큰 자체에 담긴 정보와 서명을 검증하는 것만으로 인증이 완료됩니다.

### JWT의 구조

JWT는 점(`.`)으로 구분된 세 부분으로 이루어집니다.

```
헤더.페이로드.서명
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9
.eyJzdWIiOiJ1c2VyXzEyMyIsImlhdCI6MTcyODU1NjgwMH0
.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

| 부분 | 내용 |
|---|---|
| **헤더(Header)** | 알고리즘 종류(`HS256`, `RS256` 등)와 토큰 타입 |
| **페이로드(Payload)** | 사용자 ID, 권한, 만료 시각 등 클레임(claim) |
| **서명(Signature)** | 헤더 + 페이로드를 비밀 키로 서명한 값 |

> **중요:** 페이로드는 Base64URL로 인코딩될 뿐, 암호화되지는 않습니다. 민감한 정보(비밀번호, 카드 번호 등)는 절대 포함하지 마세요.

### 액세스 토큰과 리프레시 토큰

실무에서는 두 가지 토큰을 함께 사용합니다.

- **액세스 토큰(Access Token):** 짧은 유효 기간(15분~1시간). API 요청마다 `Authorization` 헤더에 포함.
- **리프레시 토큰(Refresh Token):** 긴 유효 기간(7일~30일). 액세스 토큰 만료 시 갱신 용도로만 사용.

```javascript
// Node.js + jsonwebtoken 예시
import jwt from 'jsonwebtoken';

const ACCESS_SECRET = process.env.JWT_ACCESS_SECRET;
const REFRESH_SECRET = process.env.JWT_REFRESH_SECRET;

function generateTokens(userId) {
  const accessToken = jwt.sign(
    { sub: userId },
    ACCESS_SECRET,
    { expiresIn: '15m' }
  );
  const refreshToken = jwt.sign(
    { sub: userId },
    REFRESH_SECRET,
    { expiresIn: '7d' }
  );
  return { accessToken, refreshToken };
}

function verifyAccess(token) {
  return jwt.verify(token, ACCESS_SECRET);
}
```

---

## OAuth 2.0 이해하기

OAuth 2.0은 **제3자 앱이 사용자 리소스에 접근하도록 인가(authorization)하는 표준 프로토콜**입니다. "구글로 로그인", "GitHub으로 계속하기" 같은 소셜 로그인의 기반이 됩니다.

### 주요 역할

| 역할 | 설명 | 예시 |
|---|---|---|
| **Resource Owner** | 리소스 소유자(사용자) | 내 Gmail 계정의 주인 |
| **Client** | 접근을 요청하는 애플리케이션 | 여러분이 만드는 웹앱 |
| **Authorization Server** | 인가를 처리하는 서버 | Google, GitHub |
| **Resource Server** | 보호된 리소스를 보유한 서버 | Google API, GitHub API |

### Authorization Code Flow

가장 안전하고 널리 쓰이는 흐름입니다.

```
1. 사용자 → 클라이언트: "GitHub으로 로그인" 클릭
2. 클라이언트 → Auth Server: 인가 요청 (client_id, redirect_uri, scope)
3. Auth Server → 사용자: 로그인 + 권한 동의 화면
4. Auth Server → 클라이언트: authorization code 전달 (redirect)
5. 클라이언트 → Auth Server: code + client_secret으로 토큰 교환
6. Auth Server → 클라이언트: access_token + refresh_token 반환
7. 클라이언트 → Resource Server: access_token으로 API 호출
```

```javascript
// GitHub OAuth 예시 (Express)
app.get('/auth/github/callback', async (req, res) => {
  const { code } = req.query;

  // code를 토큰으로 교환
  const tokenRes = await fetch('https://github.com/login/oauth/access_token', {
    method: 'POST',
    headers: { Accept: 'application/json' },
    body: new URLSearchParams({
      client_id: process.env.GITHUB_CLIENT_ID,
      client_secret: process.env.GITHUB_CLIENT_SECRET,
      code,
    }),
  });
  const { access_token } = await tokenRes.json();

  // 사용자 정보 조회
  const userRes = await fetch('https://api.github.com/user', {
    headers: { Authorization: `Bearer ${access_token}` },
  });
  const user = await userRes.json();

  // 내부 JWT 발급 후 클라이언트에 전달
  const tokens = generateTokens(user.id);
  res.json(tokens);
});
```

---

## 실전 보안 패턴

### 1. 리프레시 토큰 로테이션

리프레시 토큰은 사용할 때마다 새 것으로 교체합니다. 탈취된 토큰이 사용되면 즉시 감지할 수 있습니다.

```javascript
async function refreshTokens(oldRefreshToken) {
  const payload = jwt.verify(oldRefreshToken, REFRESH_SECRET);

  // 이미 사용된 토큰이면 거부 (DB에서 확인)
  const isValid = await db.refreshTokens.findOne({
    token: oldRefreshToken,
    revoked: false,
  });
  if (!isValid) throw new Error('Token reuse detected');

  // 기존 토큰 무효화 후 새 토큰 발급
  await db.refreshTokens.updateOne(
    { token: oldRefreshToken },
    { revoked: true }
  );
  const tokens = generateTokens(payload.sub);
  await db.refreshTokens.insertOne({ token: tokens.refreshToken });
  return tokens;
}
```

### 2. 안전한 토큰 저장

클라이언트에서 토큰을 어디에 저장할지는 중요한 보안 결정입니다.

| 저장 위치 | XSS 취약성 | CSRF 취약성 | 권장 여부 |
|---|---|---|---|
| `localStorage` | 높음 | 낮음 | 비권장 |
| 메모리(변수) | 낮음 | 낮음 | 액세스 토큰에 권장 |
| `httpOnly` 쿠키 | 낮음 | 높음 | 리프레시 토큰에 권장 |

```javascript
// 리프레시 토큰은 httpOnly 쿠키로 전달
res.cookie('refreshToken', tokens.refreshToken, {
  httpOnly: true,   // JS에서 접근 불가
  secure: true,     // HTTPS에서만 전송
  sameSite: 'Strict',
  maxAge: 7 * 24 * 60 * 60 * 1000, // 7일
});

// 액세스 토큰은 응답 본문으로 전달 → 메모리에 저장
res.json({ accessToken: tokens.accessToken });
```

### 3. 미들웨어로 인증 처리

```javascript
function authenticate(req, res, next) {
  const authHeader = req.headers.authorization;
  if (!authHeader?.startsWith('Bearer ')) {
    return res.status(401).json({ error: 'Unauthorized' });
  }

  try {
    const token = authHeader.split(' ')[1];
    req.user = verifyAccess(token);
    next();
  } catch (err) {
    // 만료된 경우와 변조된 경우를 구분
    if (err.name === 'TokenExpiredError') {
      return res.status(401).json({ error: 'Token expired' });
    }
    return res.status(403).json({ error: 'Invalid token' });
  }
}

app.get('/api/me', authenticate, (req, res) => {
  res.json({ userId: req.user.sub });
});
```

---

## 정리: JWT vs 세션 중 무엇을 선택할까

두 방식 모두 장단점이 있습니다. 상황에 맞게 선택하세요.

**JWT가 유리한 경우**
- 마이크로서비스 또는 멀티 서버 환경 (공유 세션 저장소 불필요)
- 모바일 앱과 API를 함께 운용할 때
- 서버리스 환경(AWS Lambda 등)

**세션이 유리한 경우**
- 즉각적인 토큰 무효화(강제 로그아웃)가 필수인 서비스
- 전통적인 모놀리식 서버 사이드 렌더링 앱
- 저장소 관리 복잡도를 낮추고 싶을 때

인증은 보안의 핵심입니다. 표준을 따르고, 직접 암호화 알고리즘을 구현하는 대신 검증된 라이브러리를 활용하세요. 그리고 리프레시 토큰 로테이션과 `httpOnly` 쿠키 조합은 현재 실무에서 권장되는 황금 조합이니 꼭 기억해 두시기 바랍니다.
