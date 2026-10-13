---
title: WebSocket으로 실시간 기능 구현하기 — 채팅부터 알림까지 완벽 가이드
date: 2026-10-13
tags: [WebSocket, 실시간 통신, 웹 개발, Node.js, 프론트엔드]
description: WebSocket의 동작 원리를 이해하고, 브라우저와 Node.js 서버에서 채팅·알림·라이브 업데이트 기능을 구현하는 실전 방법을 소개합니다.
---

## 왜 실시간 통신이 필요한가?

기존 HTTP 요청-응답 모델은 클라이언트가 먼저 요청을 보내야만 서버에서 데이터를 받을 수 있습니다. 채팅, 실시간 알림, 주식 시세, 온라인 게임처럼 서버에서 먼저 데이터를 보내야 하는 상황에서는 이 구조가 맞지 않습니다.

과거에는 **폴링(Polling)** 으로 이 문제를 해결했습니다. 클라이언트가 주기적으로 서버에 "새로운 데이터 있어요?" 라고 물어보는 방식입니다. 하지만 이 방법은 불필요한 요청이 많고, 응답 지연이 생기며, 서버 부하가 커집니다.

**WebSocket** 은 이 문제를 근본적으로 해결합니다. 클라이언트와 서버 사이에 지속적인 양방향 연결을 만들어, 언제든 어느 쪽에서도 데이터를 보낼 수 있게 합니다.

---

## HTTP vs WebSocket 비교

| 항목 | HTTP | WebSocket |
|------|------|-----------|
| 연결 방식 | 요청마다 연결 | 한 번 연결 후 지속 |
| 통신 방향 | 단방향 (클라이언트→서버) | 양방향 |
| 오버헤드 | 헤더 반복 전송 | 최초 핸드셰이크 이후 최소화 |
| 적합한 용도 | 일반 API 호출 | 실시간 기능 |

---

## WebSocket 동작 원리

WebSocket 연결은 HTTP 업그레이드 요청으로 시작됩니다.

```
클라이언트 → 서버: GET /chat HTTP/1.1
                  Upgrade: websocket
                  Connection: Upgrade

서버 → 클라이언트: HTTP/1.1 101 Switching Protocols
                  Upgrade: websocket
                  Connection: Upgrade
```

핸드셰이크가 완료되면 TCP 연결 위에서 WebSocket 프레임 단위로 데이터를 주고받습니다. 이후에는 HTTP 헤더 없이 경량 프레임만 전송하므로 오버헤드가 매우 적습니다.

---

## Node.js 서버 구현

`ws` 라이브러리를 사용해 간단한 WebSocket 서버를 만들 수 있습니다.

```bash
npm install ws
```

```js
// server.js
const { WebSocketServer } = require('ws');

const wss = new WebSocketServer({ port: 8080 });

wss.on('connection', (ws) => {
  console.log('클라이언트 연결됨');

  ws.on('message', (data) => {
    const message = data.toString();
    console.log('받은 메시지:', message);

    // 연결된 모든 클라이언트에게 브로드캐스트
    wss.clients.forEach((client) => {
      if (client.readyState === ws.OPEN) {
        client.send(message);
      }
    });
  });

  ws.on('close', () => {
    console.log('클라이언트 연결 해제');
  });

  ws.send('서버에 연결되었습니다!');
});
```

---

## 브라우저 클라이언트 구현

브라우저에서는 별도 라이브러리 없이 내장 `WebSocket` API를 사용합니다.

```js
// client.js
const ws = new WebSocket('ws://localhost:8080');

ws.onopen = () => {
  console.log('서버에 연결됨');
  ws.send('안녕하세요, 서버!');
};

ws.onmessage = (event) => {
  console.log('받은 메시지:', event.data);
  displayMessage(event.data);
};

ws.onclose = () => {
  console.log('연결 종료됨');
};

ws.onerror = (error) => {
  console.error('WebSocket 오류:', error);
};

function sendMessage(text) {
  if (ws.readyState === WebSocket.OPEN) {
    ws.send(text);
  }
}
```

---

## Socket.IO로 더 쉽게 구현하기

`Socket.IO` 는 WebSocket을 추상화한 라이브러리로, 자동 재연결, 룸(Room), 네임스페이스 등 실무에 필요한 기능을 갖추고 있습니다.

```bash
npm install socket.io   # 서버
npm install socket.io-client  # 클라이언트
```

```js
// server.js (Socket.IO)
const { createServer } = require('http');
const { Server } = require('socket.io');

const httpServer = createServer();
const io = new Server(httpServer, {
  cors: { origin: '*' }
});

io.on('connection', (socket) => {
  console.log('사용자 접속:', socket.id);

  // 특정 방에 입장
  socket.on('join-room', (roomId) => {
    socket.join(roomId);
    socket.to(roomId).emit('user-joined', socket.id);
  });

  // 방 안의 모든 사용자에게 메시지 전송
  socket.on('chat-message', ({ roomId, message }) => {
    io.to(roomId).emit('chat-message', {
      sender: socket.id,
      message,
      timestamp: new Date().toISOString(),
    });
  });

  socket.on('disconnect', () => {
    console.log('사용자 퇴장:', socket.id);
  });
});

httpServer.listen(3000);
```

---

## 자동 재연결 구현

네트워크가 끊어졌을 때 자동으로 재연결하는 로직은 실무에서 필수입니다.

```js
function createReconnectingWebSocket(url) {
  let ws;
  let retryDelay = 1000;
  const maxDelay = 30000;

  function connect() {
    ws = new WebSocket(url);

    ws.onopen = () => {
      console.log('연결됨');
      retryDelay = 1000; // 성공하면 딜레이 초기화
    };

    ws.onclose = () => {
      console.log(`${retryDelay / 1000}초 후 재연결 시도...`);
      setTimeout(() => {
        retryDelay = Math.min(retryDelay * 2, maxDelay); // 지수 백오프
        connect();
      }, retryDelay);
    };
  }

  connect();
  return () => ws; // 현재 ws 인스턴스 반환
}
```

---

## WebSocket vs Server-Sent Events(SSE) vs 롱폴링

- **WebSocket**: 채팅, 온라인 게임, 협업 툴처럼 클라이언트↔서버 양방향 통신이 필요할 때
- **SSE(Server-Sent Events)**: 알림, 뉴스피드처럼 서버→클라이언트 단방향 스트림으로 충분할 때. 구현이 간단하고 HTTP를 그대로 활용
- **롱폴링**: 구형 브라우저 지원이 필요하거나 인프라가 WebSocket을 지원하지 않을 때의 차선책

---

## 마치며

WebSocket은 처음에는 낯설지만, 원리를 이해하면 생각보다 간단합니다. 기본 `ws` 라이브러리로 핵심 개념을 익히고, 실무에서는 자동 재연결·룸·이벤트 관리가 잘 갖춰진 Socket.IO를 활용하는 것이 좋습니다.

실시간 기능이 필요한 프로젝트라면 무조건 폴링부터 떠올리지 말고 WebSocket 도입을 먼저 검토해 보세요. 서버 부하도 줄고 사용자 경험도 크게 좋아집니다.
