---
title: JavaScript 디자인 패턴 완전 가이드 — 실무에서 자주 쓰는 4가지 핵심 패턴
date: 2026-10-01
tags: [JavaScript, 디자인 패턴, 소프트웨어 설계, 웹 개발, 코드 품질]
description: 소프트웨어 개발에서 반복되는 문제를 우아하게 해결하는 JavaScript 디자인 패턴 4가지를 실전 예제와 함께 정리했습니다.
---

디자인 패턴은 소프트웨어 개발에서 반복적으로 마주치는 문제들에 대한 검증된 해결책입니다. "바퀴를 다시 발명하지 않는다"는 원칙처럼, 잘 알려진 패턴을 적재적소에 활용하면 코드의 구조가 개선되고 팀원과의 커뮤니케이션도 훨씬 원활해집니다.

오늘은 JavaScript 실무에서 가장 자주 마주치는 4가지 디자인 패턴을 소개합니다.

## 1. 모듈 패턴 (Module Pattern)

모듈 패턴은 관련된 코드를 하나의 단위로 캡슐화하는 패턴입니다. 내부 구현을 숨기고 공개 인터페이스만 노출해 코드의 결합도를 낮출 수 있습니다.

### 기본 구현

```javascript
const counter = (() => {
  let count = 0; // private 변수

  return {
    increment() { count++; },
    decrement() { count--; },
    getCount() { return count; },
  };
})();

counter.increment();
counter.increment();
console.log(counter.getCount()); // 2
console.log(counter.count);      // undefined (외부에서 접근 불가)
```

즉시 실행 함수(IIFE)를 이용해 클로저를 만들고, 외부에 공개할 메서드만 반환합니다. `count` 변수는 반환된 객체 내부에서만 접근할 수 있어 안전하게 보호됩니다.

### 언제 사용할까?

- 전역 네임스페이스 오염을 방지하고 싶을 때
- 내부 상태를 외부로부터 보호해야 할 때
- 관련된 기능을 하나의 응집력 있는 단위로 묶고 싶을 때

## 2. 옵저버 패턴 (Observer Pattern)

옵저버 패턴은 객체의 상태 변화를 여러 구독자에게 자동으로 알리는 패턴입니다. 이벤트 시스템, Redux·MobX 같은 상태 관리 라이브러리, 리액티브 프로그래밍의 기반이 되는 패턴입니다.

### 기본 구현

```javascript
class EventEmitter {
  constructor() {
    this.events = {};
  }

  on(event, listener) {
    if (!this.events[event]) {
      this.events[event] = [];
    }
    this.events[event].push(listener);
    return this; // 메서드 체이닝 지원
  }

  emit(event, ...args) {
    if (this.events[event]) {
      this.events[event].forEach(listener => listener(...args));
    }
    return this;
  }

  off(event, listenerToRemove) {
    if (this.events[event]) {
      this.events[event] = this.events[event].filter(
        listener => listener !== listenerToRemove
      );
    }
    return this;
  }
}

// 사용 예시
const emitter = new EventEmitter();

const handleLogin = (user) => console.log(`${user.name}님이 로그인했습니다.`);
emitter.on('login', handleLogin);
emitter.emit('login', { name: '홍길동' }); // "홍길동님이 로그인했습니다."
emitter.off('login', handleLogin);          // 구독 해제
```

### 실무 활용 예시

프론트엔드에서는 컴포넌트 간 커스텀 이벤트 통신에 활용하고, 서버 사이드에서는 Node.js의 `EventEmitter`가 이 패턴을 그대로 구현하고 있습니다. React의 `useEffect` 내 이벤트 리스너 등록/해제도 같은 개념입니다.

## 3. 팩토리 패턴 (Factory Pattern)

팩토리 패턴은 객체 생성 로직을 캡슐화해 클라이언트 코드가 구체적인 클래스에 직접 의존하지 않도록 합니다. 복잡한 초기화 로직이나 조건부 객체 생성에 특히 유용합니다.

### 기본 구현

```javascript
class Dog {
  constructor(name) {
    this.name = name;
    this.type = 'dog';
  }
  speak() { return `${this.name}: 멍멍!`; }
}

class Cat {
  constructor(name) {
    this.name = name;
    this.type = 'cat';
  }
  speak() { return `${this.name}: 야옹~`; }
}

// 팩토리 함수
function createAnimal(type, name) {
  const animals = { dog: Dog, cat: Cat };
  const AnimalClass = animals[type];
  if (!AnimalClass) throw new Error(`알 수 없는 동물 타입: ${type}`);
  return new AnimalClass(name);
}

const dog = createAnimal('dog', '바둑이');
const cat = createAnimal('cat', '나비');
console.log(dog.speak()); // "바둑이: 멍멍!"
console.log(cat.speak()); // "나비: 야옹~"
```

### 언제 사용할까?

- 생성할 객체의 타입이 런타임에 결정될 때
- 복잡한 객체 생성 과정을 추상화하고 싶을 때
- 테스트 시 목(Mock) 객체를 쉽게 주입하고 싶을 때

실무에서는 API 응답 타입에 따라 다른 모델 객체를 생성하거나, 환경(개발/운영)에 따라 다른 서비스 인스턴스를 만들 때 자주 사용합니다.

## 4. 싱글톤 패턴 (Singleton Pattern)

싱글톤 패턴은 클래스의 인스턴스가 하나만 존재하도록 보장하는 패턴입니다. 데이터베이스 연결, 설정 관리, 로거(Logger) 등 앱 전체에서 하나의 상태를 공유해야 할 때 활용됩니다.

### JavaScript에서의 구현

```javascript
class Config {
  constructor() {
    if (Config.instance) {
      return Config.instance;
    }
    this.settings = {};
    Config.instance = this;
  }

  set(key, value) {
    this.settings[key] = value;
  }

  get(key) {
    return this.settings[key];
  }
}

const config1 = new Config();
const config2 = new Config();

config1.set('theme', 'dark');
console.log(config2.get('theme')); // "dark"
console.log(config1 === config2);  // true (같은 인스턴스)
```

### 주의사항

싱글톤은 전역 상태를 만들기 때문에 남용하면 코드의 테스트 가능성과 유연성이 떨어집니다. 꼭 필요한 경우에만 사용하고, 의존성 주입(DI) 패턴과 함께 활용하는 것이 좋습니다.

## 패턴 선택 가이드

| 상황 | 추천 패턴 |
|------|---------|
| 관련 기능을 묶고 내부 상태를 숨기고 싶다 | 모듈 패턴 |
| 여러 곳에 상태 변화를 자동으로 알려야 한다 | 옵저버 패턴 |
| 조건에 따라 다른 객체를 만들어야 한다 | 팩토리 패턴 |
| 앱 전체에서 하나의 인스턴스만 필요하다 | 싱글톤 패턴 |

## 마무리

디자인 패턴은 코드에 "이름"을 붙여주는 공통 언어 역할을 합니다. "여기서 옵저버 패턴 쓰면 어때요?"라고 말할 수 있게 되면 팀 내 소통이 훨씬 빨라집니다. 패턴이 목적이 아니라 수단임을 기억하면서, 오늘 소개한 4가지 패턴부터 실제 코드에 하나씩 적용해 보세요. 억지로 끼워 맞추는 것이 아니라 자연스럽게 문제를 해결하는 도구로 활용하는 것이 핵심입니다.
