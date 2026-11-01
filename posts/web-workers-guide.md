---
title: Web Workers 완전 가이드 — 브라우저에서 멀티스레딩으로 성능 극대화하기
date: 2026-11-01
tags: [Web Workers, JavaScript, 브라우저 성능, 멀티스레딩, 웹 개발]
description: Web Workers를 활용해 무거운 연산을 백그라운드 스레드로 분리하고 메인 스레드 블로킹 없이 부드러운 UI를 유지하는 방법을 소개합니다.
---

JavaScript는 싱글 스레드 언어입니다. 이 특성 때문에 무거운 계산이나 대용량 데이터 처리를 메인 스레드에서 실행하면 브라우저 UI가 멈추거나 버벅이는 현상이 생깁니다. **Web Workers**는 이 문제를 해결하는 브라우저 네이티브 API로, 별도의 백그라운드 스레드에서 JavaScript를 실행할 수 있게 해줍니다.

## Web Workers란?

Web Workers는 메인 스레드와 분리된 독립적인 JavaScript 실행 컨텍스트입니다. 워커는 DOM에 직접 접근할 수 없지만, 무거운 연산을 병렬로 처리하고 결과만 메인 스레드에 전달하는 방식으로 UI 응답성을 유지할 수 있습니다.

### 워커의 종류

- **Dedicated Worker**: 하나의 스크립트에서만 사용하는 일반적인 워커
- **Shared Worker**: 여러 탭·창에서 공유할 수 있는 워커
- **Service Worker**: 네트워크 요청을 가로채고 오프라인 지원을 담당하는 특수 워커

이 글에서는 가장 많이 쓰이는 **Dedicated Worker**를 중심으로 다룹니다.

## 기본 사용법

### 워커 생성 및 통신

메인 스레드에서 `new Worker()`로 워커를 생성하고, `postMessage()`와 `onmessage`로 데이터를 주고받습니다.

```js
// main.js
const worker = new Worker('worker.js');

worker.postMessage({ type: 'CALCULATE', data: [1, 2, 3, 4, 5] });

worker.onmessage = (event) => {
  console.log('워커 결과:', event.data);
};

worker.onerror = (error) => {
  console.error('워커 오류:', error.message);
};
```

```js
// worker.js
self.onmessage = (event) => {
  const { type, data } = event.data;

  if (type === 'CALCULATE') {
    const result = heavyCalculation(data);
    self.postMessage(result);
  }
};

function heavyCalculation(arr) {
  return arr.reduce((sum, n) => sum + n, 0);
}
```

### Inline Worker (Blob URL)

별도 파일을 만들기 번거로울 때는 Blob을 활용해 코드 내에서 워커를 생성할 수 있습니다.

```js
const workerCode = `
  self.onmessage = (e) => {
    const result = e.data * 2;
    self.postMessage(result);
  };
`;

const blob = new Blob([workerCode], { type: 'application/javascript' });
const worker = new Worker(URL.createObjectURL(blob));
```

## 실전 사용 사례

### 1. 대용량 배열 정렬·필터링

수십만 건 이상의 데이터를 프론트엔드에서 정렬하거나 필터링할 때 메인 스레드에서 처리하면 UI가 수 초간 멈출 수 있습니다. 이 작업을 워커로 분리하면 사용자는 로딩 스피너를 보면서도 다른 UI 조작이 가능합니다.

```js
// search-worker.js
self.onmessage = ({ data: { items, query } }) => {
  const filtered = items.filter(item =>
    item.title.toLowerCase().includes(query.toLowerCase())
  );
  self.postMessage(filtered);
};
```

### 2. 이미지·파일 처리

이미지 리사이징, Canvas 기반 필터 적용, CSV 파싱 등 파일을 다루는 작업도 워커에 적합합니다. `OffscreenCanvas`를 사용하면 워커 내에서 Canvas API도 활용할 수 있습니다.

```js
// image-worker.js
self.onmessage = ({ data: { imageData, width, height } }) => {
  // 그레이스케일 변환
  for (let i = 0; i < imageData.data.length; i += 4) {
    const avg = (imageData.data[i] + imageData.data[i+1] + imageData.data[i+2]) / 3;
    imageData.data[i] = imageData.data[i+1] = imageData.data[i+2] = avg;
  }
  self.postMessage(imageData, [imageData.data.buffer]);
};
```

### 3. 암호화·해싱

비밀번호 해싱이나 데이터 암호화는 CPU를 많이 사용하는 작업입니다. 워커에서 처리하면 폼 입력창이 블로킹되는 현상을 방지할 수 있습니다.

## Transferable Objects로 성능 최적화

워커와 메인 스레드 간에 데이터를 전달할 때 기본적으로 **복사(structured clone)**가 일어납니다. 대용량 `ArrayBuffer`나 이미지 데이터를 복사하면 오버헤드가 크므로, **Transferable Objects** 방식으로 소유권을 이전하면 복사 없이 O(1)에 전달할 수 있습니다.

```js
const buffer = new ArrayBuffer(1024 * 1024); // 1MB
worker.postMessage(buffer, [buffer]); // 두 번째 인자에 이전할 객체 배열 전달
// 이후 buffer는 메인 스레드에서 접근 불가
```

## WorkerPool 패턴

여러 작업을 동시에 처리하려면 워커를 미리 여러 개 생성해두는 **Pool** 패턴을 사용합니다. CPU 코어 수에 맞게 워커 수를 설정하는 것이 일반적입니다.

```js
class WorkerPool {
  constructor(size = navigator.hardwareConcurrency || 4) {
    this.workers = Array.from({ length: size }, () => new Worker('worker.js'));
    this.queue = [];
    this.idle = [...this.workers];
  }

  run(data) {
    return new Promise((resolve, reject) => {
      const task = { data, resolve, reject };
      if (this.idle.length > 0) {
        this._dispatch(task);
      } else {
        this.queue.push(task);
      }
    });
  }

  _dispatch(task) {
    const worker = this.idle.pop();
    worker.onmessage = ({ data }) => {
      task.resolve(data);
      this.idle.push(worker);
      if (this.queue.length > 0) this._dispatch(this.queue.shift());
    };
    worker.onerror = (e) => task.reject(e);
    worker.postMessage(task.data);
  }
}
```

## Web Workers의 한계

Web Workers는 강력하지만 몇 가지 제약이 있습니다.

- **DOM 접근 불가**: `document`, `window` 객체를 직접 사용할 수 없습니다.
- **localStorage 접근 제한**: 동기 API인 localStorage는 사용 불가, IndexedDB는 사용 가능합니다.
- **동일 출처 정책**: 워커 스크립트는 메인 페이지와 같은 출처여야 합니다.
- **디버깅 도구**: 브라우저 DevTools에서 워커 스레드를 별도로 디버깅할 수 있지만 익숙해지는 데 시간이 필요합니다.

## 언제 Web Workers를 써야 할까?

다음 기준에 해당하면 Web Workers 도입을 고려하세요.

1. 단일 작업이 **50ms 이상** 소요될 때 (사용자가 버벅임을 느끼는 임계값)
2. 반복적으로 수행되는 **CPU 집약적 작업**이 있을 때
3. 실시간으로 대량의 **스트리밍 데이터**를 처리해야 할 때

Web Workers는 모든 상황에 적합한 만능 해결책은 아닙니다. 통신 오버헤드와 메모리 사용량을 고려해 꼭 필요한 곳에만 적용하면, JavaScript 싱글 스레드의 한계를 넘어 훨씬 빠르고 쾌적한 웹 앱을 만들 수 있습니다.
