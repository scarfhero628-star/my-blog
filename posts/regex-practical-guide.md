---
title: 정규표현식 실전 가이드 — 개발자가 꼭 알아야 할 패턴 10가지
date: 2026-09-27
tags: [정규표현식, Regex, JavaScript, 개발자 팁, 문자열 처리]
description: 복잡한 문자열 처리를 단 몇 줄로 해결하는 정규표현식의 핵심 패턴과 실전 예제를 정리했습니다.
---

정규표현식(Regular Expression, Regex)은 처음 접하면 암호처럼 보이지만, 한 번 익혀두면 문자열 검색·검증·치환 작업을 몇 줄 만에 처리할 수 있는 강력한 도구입니다. 이 글에서는 실무에서 자주 마주치는 10가지 패턴을 예제 코드와 함께 정리합니다.

## 정규표현식이란?

정규표현식은 특정 패턴의 문자열을 찾거나 검증하기 위한 표현 언어입니다. JavaScript, Python, Java 등 거의 모든 현대 프로그래밍 언어에서 기본으로 지원합니다. 패턴을 직접 구현하면 수십 줄이 걸리는 로직도, 정규표현식 한 줄로 끝낼 수 있습니다.

```javascript
const regex = /패턴/플래그;
// 예: 이메일 검사
const emailRegex = /^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$/;
```

## 알아야 할 기본 문법

| 기호 | 의미 |
|------|------|
| `.` | 임의의 한 문자 |
| `*` | 앞 문자 0회 이상 반복 |
| `+` | 앞 문자 1회 이상 반복 |
| `?` | 앞 문자 0회 또는 1회 |
| `^` | 문자열의 시작 |
| `$` | 문자열의 끝 |
| `\d` | 숫자 (0-9) |
| `\w` | 단어 문자 (영문, 숫자, _) |
| `\s` | 공백 문자 |
| `[abc]` | a, b, c 중 하나 |
| `{n,m}` | n회 이상 m회 이하 반복 |

## 실전에서 자주 쓰는 패턴 10가지

### 1. 이메일 유효성 검사

```javascript
const emailRegex = /^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$/;

console.log(emailRegex.test('user@example.com')); // true
console.log(emailRegex.test('invalid-email'));     // false
```

### 2. 한국 전화번호

```javascript
// 010-1234-5678 또는 01012345678 형식
const phoneRegex = /^01[016789]-?\d{3,4}-?\d{4}$/;

console.log(phoneRegex.test('010-1234-5678')); // true
console.log(phoneRegex.test('01012345678'));   // true
console.log(phoneRegex.test('02-123-4567'));   // false
```

### 3. 비밀번호 강도 검사

영문, 숫자, 특수문자를 포함한 8자 이상의 비밀번호인지 확인합니다. `(?=...)` 는 긍정 전방 탐색(lookahead)으로, 해당 조건을 만족하는지만 체크하고 문자를 소비하지 않습니다.

```javascript
const passwordRegex = /^(?=.*[A-Za-z])(?=.*\d)(?=.*[@$!%*#?&])[A-Za-z\d@$!%*#?&]{8,}$/;

console.log(passwordRegex.test('Abc@1234')); // true
console.log(passwordRegex.test('abc1234'));  // false (특수문자 없음)
console.log(passwordRegex.test('Ab@12'));    // false (8자 미만)
```

### 4. URL 추출

```javascript
const urlRegex = /https?:\/\/(www\.)?[-a-zA-Z0-9@:%._+~#=]{1,256}\.[a-zA-Z0-9()]{1,6}\b([-a-zA-Z0-9()@:%_+.~#?&/=]*)/gi;

const text = '블로그 주소는 https://example.com이고, 참고 자료는 https://docs.dev/guide 입니다.';
const urls = text.match(urlRegex);
console.log(urls); // ['https://example.com', 'https://docs.dev/guide']
```

### 5. 공백 정규화

연속된 공백이나 탭을 단일 공백으로 치환합니다. 사용자 입력을 정리할 때 자주 사용합니다.

```javascript
const normalize = (str) => str.replace(/\s+/g, ' ').trim();

console.log(normalize('  Hello    World  ')); // 'Hello World'
console.log(normalize('\t이름:\t홍길동  ')); // '이름: 홍길동'
```

### 6. 한글만 추출

```javascript
const koreanRegex = /[가-힣]+/g;

const text = 'Hello 안녕하세요 World 반갑습니다';
const korean = text.match(koreanRegex);
console.log(korean); // ['안녕하세요', '반갑습니다']
```

유니코드 범위 `[가-힣]`은 현대 한글 완성형 글자 전체를 포함합니다.

### 7. HTML 태그 제거

```javascript
const stripHtml = (html) => html.replace(/<[^>]*>/g, '');

console.log(stripHtml('<p>Hello <b>World</b></p>')); // 'Hello World'
console.log(stripHtml('<a href="/link">클릭</a>'));   // '클릭'
```

### 8. 날짜 형식 검사 (YYYY-MM-DD)

```javascript
const dateRegex = /^\d{4}-(0[1-9]|1[0-2])-(0[1-9]|[12]\d|3[01])$/;

console.log(dateRegex.test('2026-09-27')); // true
console.log(dateRegex.test('2026-13-01')); // false (월이 13)
console.log(dateRegex.test('2026-9-27'));  // false (월이 한 자리)
```

### 9. 카멜케이스 변환

```javascript
const toCamelCase = (str) =>
  str.replace(/-([a-z])/g, (_, char) => char.toUpperCase());

console.log(toCamelCase('my-variable-name')); // 'myVariableName'
console.log(toCamelCase('background-color')); // 'backgroundColor'
```

CSS 프로퍼티나 REST API의 케밥케이스(kebab-case)를 JavaScript 변수명으로 변환할 때 유용합니다.

### 10. 민감 정보 마스킹

```javascript
// 신용카드 번호: 앞 4자리, 뒤 4자리만 남기고 나머지 마스킹
const maskCard = (number) =>
  number.replace(/(\d{4})\d{8}(\d{4})/, '$1-****-****-$2');

// 이름: 두 번째 글자부터 끝에서 두 번째까지 마스킹
const maskName = (name) =>
  name.replace(/(?<=.).(?=.)/, '*');

console.log(maskCard('1234567890123456')); // '1234-****-****-3456'
console.log(maskName('홍길동'));           // '홍*동'
```

## 유용한 플래그

| 플래그 | 설명 |
|--------|------|
| `g` | 전체 문자열에서 모든 일치 항목 찾기 |
| `i` | 대소문자 구분 없이 검색 |
| `m` | 각 줄의 시작/끝에 `^`, `$` 적용 |
| `s` | `.`이 줄바꿈 문자까지 포함 |

```javascript
// 대소문자 구분 없이 전체 검색
const matches = 'Hello hello HELLO'.match(/hello/gi);
console.log(matches); // ['Hello', 'hello', 'HELLO']

// 멀티라인: 각 줄의 시작에서 패턴 매칭
const lines = 'first\nsecond\nthird'.match(/^\w+/gm);
console.log(lines); // ['first', 'second', 'third']
```

## 정규표현식 성능 팁

- **중첩 수량자는 피하세요.** `(.+)+` 같은 패턴은 catastrophic backtracking을 일으켜 실행 시간이 기하급수적으로 늘어납니다.
- **반복 사용하는 정규표현식은 변수에 저장하세요.** 함수 내부에서 `/패턴/` 리터럴을 반복 선언하면 매번 컴파일 비용이 발생합니다.
- **`test()`는 `match()`보다 빠릅니다.** 일치 여부만 확인할 때는 `exec()` 나 `match()` 대신 `test()`를 사용하세요.
- **원자 그룹 또는 소유적 수량자를 활용하세요.** JavaScript ES2018 이후 지원되는 원자 그룹 `(?>...)` 또는 소유적 수량자 `++`, `*+` 을 사용하면 역추적을 방지할 수 있습니다.

## 마치며

정규표현식은 처음에는 낯설어 보이지만, 기본 기호 몇 가지만 익히면 실무에서 놀라운 위력을 발휘합니다. 패턴을 외우려 하지 말고, 필요할 때마다 참고하면서 자신만의 패턴 스니펫 라이브러리를 만들어보세요.

연습하고 싶다면 [regex101.com](https://regex101.com)을 활용해 보세요. 패턴을 실시간으로 테스트하고 각 부분의 의미를 자세히 설명해줘서 학습에 큰 도움이 됩니다.
