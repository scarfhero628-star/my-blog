---
title: WebAssembly 입문 가이드 — 브라우저에서 C/C++·Rust 코드를 실행하는 혁신 기술
date: 2026-11-24
tags: [WebAssembly, WASM, 웹 성능, Rust, 브라우저]
description: WebAssembly의 핵심 개념과 동작 원리를 이해하고, JavaScript와 함께 고성능 웹 애플리케이션을 구축하는 실전 방법을 소개합니다.
---

## WebAssembly란 무엇인가?

WebAssembly(이하 WASM)는 2019년 W3C 공식 표준으로 채택된 바이너리 명령어 형식입니다. 웹 브라우저에서 C, C++, Rust 등 다양한 언어로 작성된 코드를 **네이티브에 가까운 속도**로 실행할 수 있게 해줍니다.

기존 JavaScript만으로는 한계가 있었던 연산 집약적인 작업—이미지 처리, 3D 렌더링, 암호화, 비디오 편집—에서 WASM은 획기적인 성능 향상을 가져옵니다.

> "WebAssembly는 JavaScript를 대체하는 것이 아니라 **보완**합니다. 두 기술은 서로 긴밀히 협력합니다."

---

## WebAssembly의 핵심 특징

### 1. 빠른 실행 속도

WASM은 컴파일된 바이너리 형식이기 때문에 JavaScript처럼 파싱(parse)하고 컴파일하는 과정이 훨씬 짧습니다.

| 항목 | JavaScript | WebAssembly |
|------|-----------|-------------|
| 파싱 | 느림 (텍스트 → AST) | 빠름 (바이너리 직접 디코딩) |
| 컴파일 | JIT 컴파일 | AOT 사전 컴파일 가능 |
| 실행 속도 | 빠름 | 네이티브의 80~90% |
| 타입 | 동적 | 정적 (Int32, Float64 등) |

### 2. 안전한 샌드박스 실행

WASM은 브라우저의 보안 샌드박스 안에서 실행됩니다. 직접 DOM이나 Web API에 접근할 수 없으며, 반드시 JavaScript를 통해 통신해야 합니다. 이 제약이 오히려 보안을 강화합니다.

### 3. 이식성과 언어 중립성

WASM을 타겟으로 컴파일할 수 있는 언어는 계속 늘고 있습니다.

- **C / C++** — Emscripten 툴체인
- **Rust** — wasm-pack, wasm-bindgen
- **Go** — 공식 WASM 지원
- **AssemblyScript** — TypeScript 유사 문법으로 직접 WASM 작성
- **Python** — Pyodide 프로젝트

---

## 동작 원리 이해하기

WASM 모듈이 브라우저에서 실행되는 과정은 다음과 같습니다.

```
소스 코드 (Rust/C++) → 컴파일 → .wasm 바이너리
                                      ↓
브라우저 → fetch('.wasm') → WebAssembly.instantiate() → 모듈 인스턴스
                                                              ↓
                                                    JavaScript에서 함수 호출
```

### WAT (WebAssembly Text Format)

`.wasm`은 사람이 읽기 어렵지만, **WAT(WebAssembly Text Format)**로 변환하면 내부 구조를 이해할 수 있습니다.

```wat
;; 두 정수를 더하는 간단한 함수
(module
  (func $add (param $a i32) (param $b i32) (result i32)
    local.get $a
    local.get $b
    i32.add)
  (export "add" (func $add))
)
```

---

## Rust로 WebAssembly 모듈 만들기

Rust는 WASM 개발에 가장 잘 지원되는 언어입니다. `wasm-pack`을 사용하면 npm 패키지처럼 배포할 수 있습니다.

### 환경 설정

```bash
# Rust 설치 (이미 있으면 생략)
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# WASM 타겟 추가
rustup target add wasm32-unknown-unknown

# wasm-pack 설치
cargo install wasm-pack
```

### Rust 코드 작성

```rust
// src/lib.rs
use wasm_bindgen::prelude::*;

// JavaScript에서 호출 가능하도록 export
#[wasm_bindgen]
pub fn fibonacci(n: u32) -> u32 {
    if n <= 1 {
        return n;
    }
    fibonacci(n - 1) + fibonacci(n - 2)
}

#[wasm_bindgen]
pub fn greet(name: &str) -> String {
    format!("안녕하세요, {}!", name)
}
```

### 빌드 및 패키징

```bash
wasm-pack build --target web
# pkg/ 디렉토리에 .wasm 파일과 JS 바인딩 코드가 생성됩니다
```

### JavaScript에서 호출

```javascript
import init, { fibonacci, greet } from './pkg/my_wasm.js';

async function main() {
  // WASM 모듈 초기화
  await init();

  // Rust 함수 호출
  console.log(fibonacci(40));       // 102334155
  console.log(greet('WebAssembly')); // 안녕하세요, WebAssembly!
}

main();
```

---

## 실전 활용 사례

### 이미지·영상 처리

FFmpeg를 WASM으로 컴파일한 **ffmpeg.wasm** 프로젝트는 브라우저에서 서버 없이 비디오를 변환합니다.

```javascript
import { createFFmpeg, fetchFile } from '@ffmpeg/ffmpeg';

const ffmpeg = createFFmpeg({ log: true });

async function convertVideo(videoFile) {
  await ffmpeg.load();
  ffmpeg.FS('writeFile', 'input.mp4', await fetchFile(videoFile));
  await ffmpeg.run('-i', 'input.mp4', 'output.gif');
  const data = ffmpeg.FS('readFile', 'output.gif');
  return URL.createObjectURL(new Blob([data.buffer], { type: 'image/gif' }));
}
```

### 암호화 연산

복잡한 암호화 알고리즘을 Rust로 구현하고 WASM으로 배포하면, 브라우저에서 고성능 암호화를 클라이언트 측에서 처리할 수 있습니다.

### 게임 엔진

Unity, Unreal Engine 모두 WebGL과 함께 WASM 익스포트를 지원합니다. **Figma**가 C++로 작성된 렌더링 엔진을 WASM으로 컴파일한 대표적인 사례입니다.

---

## JavaScript와 WASM 간 데이터 통신

WASM 모듈은 숫자형(i32, i64, f32, f64)만 직접 주고받을 수 있습니다. 문자열이나 배열 등 복잡한 데이터는 **선형 메모리(Linear Memory)** 를 통해 공유합니다.

```javascript
// WASM 선형 메모리에 직접 접근
const memory = new WebAssembly.Memory({ initial: 1 }); // 64KB 페이지 1개
const buffer = new Uint8Array(memory.buffer);

// 문자열을 메모리에 써서 WASM에 전달
const encoder = new TextEncoder();
const bytes = encoder.encode('Hello WASM');
buffer.set(bytes, 0); // 오프셋 0에 기록
```

`wasm-bindgen`은 이 복잡한 메모리 관리를 자동으로 처리해주기 때문에, Rust + wasm-bindgen 조합에서는 문자열이나 슬라이스도 편리하게 주고받을 수 있습니다.

---

## WASM 성능이 항상 더 빠른 것은 아닙니다

WASM을 맹신하면 안 됩니다. 주의해야 할 점들이 있습니다.

- **JavaScript ↔ WASM 경계 비용**: 호출이 잦으면 오버헤드가 쌓입니다. 한 번에 큰 연산을 넘기는 편이 낫습니다.
- **DOM 접근 불가**: UI 조작은 여전히 JavaScript 몫입니다.
- **로딩 시간**: `.wasm` 파일을 별도로 내려받아야 합니다. 작은 작업에는 오히려 불리합니다.
- **개발 복잡도**: 빌드 파이프라인과 디버깅이 훨씬 복잡해집니다.

### 언제 WASM을 쓸까?

| 적합한 경우 | 부적합한 경우 |
|------------|-------------|
| 수치 계산, 시뮬레이션 | 간단한 DOM 조작 |
| 이미지·오디오·비디오 처리 | 주로 네트워크 I/O 위주의 작업 |
| 기존 C/C++/Rust 라이브러리 재사용 | 개발 속도가 최우선인 프로토타입 |
| 게임, 3D 렌더링 | 소규모 유틸리티 함수 |

---

## WASI — 브라우저 밖에서의 WebAssembly

**WASI(WebAssembly System Interface)**는 WASM을 서버·CLI·에지 환경에서도 실행하기 위한 표준입니다. Node.js, Deno, Wasmtime 같은 런타임에서 지원합니다.

WASI의 비전은 단 하나입니다.

> "한 번 컴파일하면 어디서든 실행한다 — 브라우저, 서버, 에지, IoT 장치까지."

Cloudflare Workers, Fastly Compute, Fermyon Spin 같은 에지 플랫폼이 이미 WASM/WASI를 프로덕션에서 활용하고 있습니다.

---

## 마치며

WebAssembly는 웹의 영역을 비약적으로 확장하고 있습니다. 10년 전에는 브라우저에서 고성능 게임 엔진이나 비디오 편집기를 실행한다는 것이 꿈 같은 얘기였지만, 지금은 현실입니다.

JavaScript 개발자라면 당장 WASM으로 전환할 필요는 없지만, **성능 병목이 생겼을 때 꺼낼 수 있는 강력한 카드**로 알아두면 분명 도움이 됩니다. Rust 학습을 겸해 간단한 WASM 모듈을 만들어 보는 것을 추천합니다.
