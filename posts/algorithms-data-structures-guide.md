---
title: 알고리즘과 자료구조 입문 — 코딩 테스트와 실무를 동시에 잡는 핵심 개념 정리
date: 2026-10-18
tags: [알고리즘, 자료구조, 코딩 테스트, 컴퓨터 과학, 개발자]
description: 개발자라면 꼭 알아야 할 핵심 자료구조와 알고리즘을 실전 예제와 함께 정리합니다.
---

알고리즘과 자료구조는 개발자 채용 면접의 단골 주제이자, 실무에서 성능 좋은 코드를 작성하기 위한 기초 체력입니다. 하지만 처음 접하면 어디서부터 시작해야 할지 막막하게 느껴지기도 합니다. 이 글에서는 코딩 테스트와 실무에서 가장 자주 등장하는 핵심 자료구조와 알고리즘을 JavaScript 예제와 함께 정리합니다.

---

## 왜 알고리즘과 자료구조를 배워야 하는가?

프레임워크와 라이브러리는 빠르게 변하지만, 알고리즘과 자료구조는 수십 년째 변하지 않는 컴퓨터 과학의 근간입니다. 이를 이해하면 다음과 같은 이점이 생깁니다.

- **성능 개선**: 같은 결과를 O(n²) 대신 O(n log n)으로 처리할 수 있습니다.
- **문제 해결 능력 향상**: 복잡한 비즈니스 로직을 체계적으로 분해하는 눈이 생깁니다.
- **면접 통과율 상승**: FAANG을 포함한 대부분의 기술 기업이 코딩 테스트를 요구합니다.
- **코드 리뷰 역량 강화**: 동료의 코드에서 비효율적인 구조를 발견하고 개선안을 제시할 수 있습니다.

---

## 1. 시간 복잡도와 빅-O 표기법

알고리즘을 배우기 전에 먼저 성능을 측정하는 언어를 이해해야 합니다. 빅-O 표기법은 입력 크기가 커질 때 알고리즘이 얼마나 느려지는지를 나타냅니다.

| 표기 | 이름 | 예시 |
|---|---|---|
| O(1) | 상수 시간 | 배열 인덱스 접근 |
| O(log n) | 로그 시간 | 이진 탐색 |
| O(n) | 선형 시간 | 배열 순회 |
| O(n log n) | 선형 로그 시간 | 병합 정렬 |
| O(n²) | 이차 시간 | 버블 정렬 |

실무에서는 O(n²) 이상의 복잡도가 나오면 일단 최적화를 고민하는 습관을 들이는 게 좋습니다.

---

## 2. 핵심 자료구조

### 배열과 해시맵

가장 기본적이면서도 가장 많이 쓰이는 자료구조입니다. 특히 해시맵(JavaScript의 `Map` 또는 객체)은 두 수의 합처럼 O(n²)짜리 문제를 O(n)으로 줄이는 핵심 도구입니다.

```javascript
// 두 수의 합 — 해시맵으로 O(n) 풀기
function twoSum(nums, target) {
  const seen = new Map();
  for (let i = 0; i < nums.length; i++) {
    const complement = target - nums[i];
    if (seen.has(complement)) {
      return [seen.get(complement), i];
    }
    seen.set(nums[i], i);
  }
  return [];
}
```

### 스택과 큐

스택(Stack)은 **LIFO(후입선출)**, 큐(Queue)는 **FIFO(선입선출)** 구조입니다. 괄호 검사, BFS/DFS, 브라우저 히스토리 같은 곳에 쓰입니다.

```javascript
// 유효한 괄호 검사 — 스택 활용
function isValid(s) {
  const stack = [];
  const map = { ')': '(', '}': '{', ']': '[' };
  for (const ch of s) {
    if ('({['.includes(ch)) {
      stack.push(ch);
    } else {
      if (stack.pop() !== map[ch]) return false;
    }
  }
  return stack.length === 0;
}
```

### 연결 리스트

배열과 달리 연속된 메모리를 사용하지 않아 삽입·삭제가 O(1)입니다. 면접에서는 **역순 뒤집기**, **사이클 감지**, **두 리스트 병합** 문제가 자주 나옵니다.

```javascript
class ListNode {
  constructor(val, next = null) {
    this.val = val;
    this.next = next;
  }
}

// 연결 리스트 역순 뒤집기
function reverseList(head) {
  let prev = null;
  let curr = head;
  while (curr) {
    const next = curr.next;
    curr.next = prev;
    prev = curr;
    curr = next;
  }
  return prev;
}
```

---

## 3. 트리와 그래프

### 이진 탐색 트리(BST)

정렬된 데이터를 효율적으로 탐색하기 위한 구조입니다. 삽입·탐색·삭제가 평균 O(log n)입니다.

```javascript
// BST 탐색
function searchBST(root, val) {
  if (!root) return null;
  if (root.val === val) return root;
  return val < root.val
    ? searchBST(root.left, val)
    : searchBST(root.right, val);
}
```

### DFS와 BFS

그래프 탐색의 양대 산맥입니다. **DFS(깊이 우선 탐색)**은 재귀 또는 스택으로, **BFS(너비 우선 탐색)**은 큐로 구현합니다.

```javascript
// 트리의 최대 깊이 — DFS
function maxDepth(root) {
  if (!root) return 0;
  return 1 + Math.max(maxDepth(root.left), maxDepth(root.right));
}

// 레벨 순회 — BFS
function levelOrder(root) {
  if (!root) return [];
  const result = [];
  const queue = [root];
  while (queue.length) {
    const level = [];
    const size = queue.length;
    for (let i = 0; i < size; i++) {
      const node = queue.shift();
      level.push(node.val);
      if (node.left) queue.push(node.left);
      if (node.right) queue.push(node.right);
    }
    result.push(level);
  }
  return result;
}
```

---

## 4. 정렬 알고리즘

### 실무에서 중요한 이유

JavaScript의 `Array.prototype.sort()`는 내부적으로 TimSort를 사용하지만, 알고리즘의 원리를 이해하면 커스텀 정렬이나 부분 정렬 문제를 더 효과적으로 해결할 수 있습니다.

```javascript
// 병합 정렬 — O(n log n) 보장
function mergeSort(arr) {
  if (arr.length <= 1) return arr;
  const mid = Math.floor(arr.length / 2);
  const left = mergeSort(arr.slice(0, mid));
  const right = mergeSort(arr.slice(mid));
  return merge(left, right);
}

function merge(left, right) {
  const result = [];
  let i = 0, j = 0;
  while (i < left.length && j < right.length) {
    if (left[i] <= right[j]) result.push(left[i++]);
    else result.push(right[j++]);
  }
  return result.concat(left.slice(i)).concat(right.slice(j));
}
```

---

## 5. 이진 탐색

정렬된 배열에서 O(log n)으로 값을 찾는 기법입니다. 단순 탐색뿐 아니라 **"조건을 만족하는 최솟값/최댓값 찾기"** 유형에도 강력하게 활용됩니다.

```javascript
// 기본 이진 탐색
function binarySearch(nums, target) {
  let left = 0, right = nums.length - 1;
  while (left <= right) {
    const mid = Math.floor((left + right) / 2);
    if (nums[mid] === target) return mid;
    if (nums[mid] < target) left = mid + 1;
    else right = mid - 1;
  }
  return -1;
}
```

---

## 6. 동적 프로그래밍 (DP)

복잡한 문제를 작은 하위 문제로 나누고, 중복 계산을 메모이제이션으로 줄이는 기법입니다. 처음에는 어렵게 느껴지지만 패턴을 익히면 빠르게 적용할 수 있습니다.

```javascript
// 피보나치 — 메모이제이션으로 O(n) 구현
function fib(n, memo = {}) {
  if (n in memo) return memo[n];
  if (n <= 1) return n;
  memo[n] = fib(n - 1, memo) + fib(n - 2, memo);
  return memo[n];
}

// 계단 오르기 — 바텀업 DP
function climbStairs(n) {
  if (n <= 2) return n;
  const dp = [0, 1, 2];
  for (let i = 3; i <= n; i++) {
    dp[i] = dp[i - 1] + dp[i - 2];
  }
  return dp[n];
}
```

---

## 7. 실전 학습 전략

### 로드맵

1. **기초 다지기** (2~3주): 배열, 해시맵, 문자열 문제 집중 풀이
2. **트리·그래프** (3~4주): BFS/DFS 패턴 체화
3. **고급 주제** (4~6주): DP, 그리디, 투 포인터, 슬라이딩 윈도우
4. **반복 복습**: 틀린 문제는 3일 후, 1주 후에 다시 풀기

### 추천 연습 순서

- LeetCode Easy → Medium 순으로 풀기
- 문제 유형별 묶어서 학습 (해시맵 20문제 → 스택 20문제 → ...)
- 시간을 재고 풀기 (실제 면접과 동일한 환경)

### 흔한 실수

- 알고리즘 코드를 **이해 없이 암기**하는 것
- 정답을 보고 넘어가는 것 (반드시 직접 다시 구현해 보기)
- 시간 복잡도를 분석하지 않고 제출하는 것

---

## 마치며

알고리즘은 단기간에 마스터할 수 없지만, 꾸준한 반복으로 분명히 느는 분야입니다. 처음에는 Easy 문제도 막막하게 느껴질 수 있지만, 100문제를 풀고 나면 패턴이 보이기 시작합니다. 중요한 것은 **매일 조금씩, 포기하지 않는 것**입니다. 오늘 한 문제부터 시작해 보세요.
