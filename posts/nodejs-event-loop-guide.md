---
title: Node.js 이벤트 루프 완전 가이드 — 비동기 처리의 비밀을 파헤치다
date: 2026-10-22
tags: [Node.js, 이벤트 루프, 비동기, JavaScript, 백엔드]
description: Node.js의 이벤트 루프 동작 원리를 단계별로 파헤쳐, 비동기 코드를 예측 가능하게 작성하는 방법을 소개합니다.
---

## Node.js가 빠른 이유

Node.js는 단일 스레드임에도 수천 개의 동시 요청을 처리할 수 있습니다. 그 비결이 바로 **이벤트 루프(Event Loop)**입니다. 이벤트 루프를 이해하면 비동기 코드가 왜 그 순서로 실행되는지, 왜 특정 코드가 예상과 다르게 동작하는지 명확하게 파악할 수 있습니다.

## 이벤트 루프의 구조

이벤트 루프는 여러 단계(Phase)로 구성됩니다. 각 단계는 특정 종류의 콜백을 처리합니다.

```
   ┌───────────────────────────┐
┌─>│           timers          │  setTimeout, setInterval
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │     pending callbacks     │  I/O 에러 콜백
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │       idle, prepare       │  내부 전용
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │           poll            │  I/O 이벤트 처리
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │           check           │  setImmediate
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
└──┤      close callbacks      │  소켓 종료 등
   └───────────────────────────┘
```

### 주요 단계 설명

- **timers**: `setTimeout`과 `setInterval`의 콜백이 실행됩니다. 지정한 시간이 지난 콜백만 실행하며, 정확한 시간을 보장하지는 않습니다.
- **poll**: 새로운 I/O 이벤트를 가져와 처리합니다. 파일 읽기, 네트워크 요청 완료 등의 콜백이 여기서 실행됩니다.
- **check**: `setImmediate`의 콜백이 실행됩니다.

## 마이크로태스크 큐

이벤트 루프의 각 단계 사이에는 **마이크로태스크 큐**가 실행됩니다. 여기에 해당하는 것들은:

1. `process.nextTick()` 콜백
2. `Promise` 해결/거부 콜백 (`.then`, `.catch`)

마이크로태스크는 다음 이벤트 루프 단계로 넘어가기 전에 **모두** 처리됩니다. 이 점이 `setTimeout`이나 `setImmediate`와 근본적으로 다릅니다.

### 실행 순서 예제

```javascript
console.log('1: 동기 코드');

setTimeout(() => console.log('2: setTimeout'), 0);

Promise.resolve().then(() => console.log('3: Promise'));

process.nextTick(() => console.log('4: nextTick'));

console.log('5: 동기 코드 끝');
```

출력 결과:

```
1: 동기 코드
5: 동기 코드 끝
4: nextTick
3: Promise
2: setTimeout
```

`process.nextTick`이 `Promise`보다 먼저, 그리고 `setTimeout`보다 훨씬 먼저 실행됩니다. 콜 스택이 비워진 직후 마이크로태스크 큐가 모두 소진될 때까지 처리되기 때문입니다.

## setImmediate vs setTimeout

같은 컨텍스트에서 실행하면 `setImmediate`와 `setTimeout(fn, 0)`의 실행 순서는 보장되지 않습니다. 운영체제 타이머 오차에 따라 달라질 수 있기 때문입니다. 하지만 I/O 콜백 내부에서는 항상 `setImmediate`가 먼저 실행됩니다.

```javascript
const fs = require('fs');

fs.readFile(__filename, () => {
  setTimeout(() => console.log('setTimeout'), 0);
  setImmediate(() => console.log('setImmediate'));
});
// 항상: setImmediate → setTimeout
```

I/O 콜백은 poll 단계에서 실행되고, 그 직후 check 단계(setImmediate)가 오기 때문입니다. timers 단계는 한 바퀴 뒤에 오므로 `setTimeout`이 나중에 실행됩니다.

## 이벤트 루프 막지 않기

Node.js에서 가장 중요한 규칙 중 하나는 **이벤트 루프를 블로킹하지 않는 것**입니다. 동기 작업이 오래 실행되면 그 시간 동안 다른 모든 요청이 대기 상태에 놓입니다.

### 잘못된 예시

```javascript
// 이벤트 루프를 막는 동기 작업
app.get('/users', (req, res) => {
  const data = fs.readFileSync('users.json'); // ❌ 블로킹
  res.json(JSON.parse(data));
});
```

### 올바른 예시

```javascript
// 비동기로 처리
app.get('/users', async (req, res) => {
  const data = await fs.promises.readFile('users.json'); // ✅ 논블로킹
  res.json(JSON.parse(data));
});
```

동기 파일 읽기(`readFileSync`)는 파일이 다 읽힐 때까지 이벤트 루프를 멈춰버립니다. 그 사이에 들어오는 모든 요청은 대기해야 합니다.

## CPU 집약적 작업 처리

암호화, 이미지 처리, 대규모 데이터 연산 같은 CPU 집약적 작업은 이벤트 루프를 오래 블로킹합니다. 이런 경우 **Worker Threads**를 사용해 별도 스레드에서 처리하세요.

```javascript
const { Worker, isMainThread, parentPort } = require('worker_threads');

if (isMainThread) {
  const worker = new Worker(__filename);
  worker.on('message', (result) => console.log('결과:', result));
  worker.postMessage({ data: 'large dataset' });
} else {
  parentPort.on('message', ({ data }) => {
    // 무거운 계산을 별도 스레드에서 처리
    const result = heavyComputation(data);
    parentPort.postMessage(result);
  });
}
```

Worker Thread를 사용하면 메인 이벤트 루프는 계속 다른 요청을 처리할 수 있고, 무거운 작업은 별도 스레드에서 처리됩니다.

## 이벤트 루프 모니터링

운영 환경에서 이벤트 루프 지연을 감지하려면 `perf_hooks`나 외부 라이브러리를 사용하세요.

```javascript
const { monitorEventLoopDelay } = require('perf_hooks');

const monitor = monitorEventLoopDelay({ resolution: 10 });
monitor.enable();

setInterval(() => {
  console.log('이벤트 루프 지연(p99):', monitor.percentile(99), 'ms');
}, 5000);
```

p99가 지속적으로 높다면 어딘가에서 이벤트 루프가 블로킹되고 있다는 신호입니다.

## 정리

이벤트 루프를 이해하면 다음과 같은 이점이 있습니다.

- **디버깅**: 예상치 못한 실행 순서를 설명할 수 있습니다.
- **성능**: 이벤트 루프를 막는 코드를 식별하고 수정할 수 있습니다.
- **설계**: 비동기 코드를 더 예측 가능하게 작성할 수 있습니다.

Node.js의 강점은 I/O 집약적인 작업을 효율적으로 처리하는 데 있습니다. 이벤트 루프의 동작 방식을 존중하는 코드를 작성할 때 Node.js는 비로소 최고의 성능을 발휘합니다. 동기 코드를 최소화하고, CPU 집약적인 작업은 Worker Threads로 분리하고, 마이크로태스크 큐의 우선순위를 이해하는 것이 Node.js 개발자의 핵심 역량입니다.
