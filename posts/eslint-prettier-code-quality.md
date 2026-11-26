---
title: ESLint과 Prettier로 코드 품질 자동화하기 — 일관된 코드 스타일을 팀 전체에 적용하는 방법
date: 2026-11-26
tags: [ESLint, Prettier, 코드 품질, JavaScript, TypeScript]
description: ESLint과 Prettier를 프로젝트에 도입해 코드 스타일을 자동으로 통일하고 잠재적 버그를 사전에 잡는 실전 설정 방법을 소개합니다.
---

## 코드 스타일 논쟁을 끝내는 방법

팀 프로젝트에서 한 번쯤 겪어봤을 장면이 있습니다. 코드 리뷰에서 "세미콜론 붙여주세요", "들여쓰기 탭이 아니라 스페이스로요", "화살표 함수로 바꿔주세요" 같은 스타일 지적이 반복되는 상황입니다. 이런 댓글은 로직이 아닌 형식에 집중하게 만들고, 리뷰어와 작성자 모두의 시간을 낭비합니다.

**ESLint**과 **Prettier**는 이 문제를 도구로 해결합니다. 스타일 논쟁을 사람 간의 협의 대신 설정 파일 한 줄로 종결짓고, 포매팅과 품질 검사를 저장 시점에 자동으로 처리합니다.

---

## ESLint vs Prettier — 역할이 다르다

두 도구는 종종 함께 언급되지만 역할이 명확하게 다릅니다.

| 도구 | 역할 |
|------|------|
| **ESLint** | 코드의 **문제를 찾는다** (미사용 변수, 잘못된 비교, 접근 불가 코드 등) |
| **Prettier** | 코드를 **예쁘게 포매팅한다** (들여쓰기, 줄 길이, 따옴표 종류 등) |

ESLint는 "이 코드에 버그가 있을 수 있다"를 알려주는 **린터(linter)**이고, Prettier는 "코드 모양을 이렇게 고쳐라"를 실행하는 **포매터(formatter)**입니다. 둘은 충돌할 수 있기 때문에 함께 사용할 때는 설정이 필요합니다.

---

## ESLint 설치 및 기본 설정

```bash
npm init @eslint/config
```

인터랙티브 질문에 답하면 `.eslintrc.js`(또는 `.eslintrc.json`)가 생성됩니다. 직접 설치하려면:

```bash
npm install --save-dev eslint
npx eslint --init
```

기본 설정 파일 예시:

```js
// .eslintrc.js
module.exports = {
  env: {
    browser: true,
    es2021: true,
    node: true,
  },
  extends: ['eslint:recommended'],
  parserOptions: {
    ecmaVersion: 'latest',
    sourceType: 'module',
  },
  rules: {
    'no-unused-vars': 'warn',
    'no-console': 'warn',
    'eqeqeq': 'error',
  },
};
```

### 주요 규칙 종류

- `'off'` / `0` — 규칙 비활성화
- `'warn'` / `1` — 경고 (빌드는 통과)
- `'error'` / `2` — 에러 (빌드 실패)

---

## Prettier 설치 및 기본 설정

```bash
npm install --save-dev prettier
```

`.prettierrc` 파일을 프로젝트 루트에 생성합니다:

```json
{
  "semi": true,
  "singleQuote": true,
  "tabWidth": 2,
  "trailingComma": "es5",
  "printWidth": 100,
  "arrowParens": "always"
}
```

특정 파일을 포매팅에서 제외하려면 `.prettierignore`를 사용합니다:

```
node_modules
dist
build
*.min.js
```

---

## ESLint과 Prettier 함께 사용하기

ESLint에는 포매팅 관련 규칙도 있어서 Prettier와 충돌할 수 있습니다. 이를 해결하는 공식 방법은 `eslint-config-prettier`를 사용하는 것입니다.

```bash
npm install --save-dev eslint-config-prettier
```

`.eslintrc.js`의 `extends` 배열 맨 끝에 추가합니다:

```js
extends: [
  'eslint:recommended',
  'prettier', // 반드시 마지막에 위치해야 한다
],
```

이렇게 하면 ESLint의 포매팅 규칙이 비활성화되어 Prettier가 단독으로 포매팅을 담당합니다.

---

## TypeScript 프로젝트 설정

TypeScript를 사용한다면 추가 패키지가 필요합니다:

```bash
npm install --save-dev @typescript-eslint/parser @typescript-eslint/eslint-plugin
```

```js
// .eslintrc.js (TypeScript 프로젝트)
module.exports = {
  parser: '@typescript-eslint/parser',
  extends: [
    'eslint:recommended',
    'plugin:@typescript-eslint/recommended',
    'prettier',
  ],
  plugins: ['@typescript-eslint'],
  rules: {
    '@typescript-eslint/no-explicit-any': 'warn',
    '@typescript-eslint/explicit-function-return-type': 'off',
  },
};
```

---

## VS Code에서 저장 시 자동 적용

`.vscode/settings.json`을 프로젝트에 추가하면 팀 전체가 동일한 에디터 설정을 공유할 수 있습니다:

```json
{
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": true
  }
}
```

필요한 VS Code 확장:
- **ESLint** (`dbaeumer.vscode-eslint`)
- **Prettier - Code formatter** (`esbenp.prettier-vscode`)

---

## Git Hooks로 커밋 전 자동 검사

저장 시 자동 포매팅도 좋지만, 팀원 전체가 같은 에디터 설정을 쓴다는 보장은 없습니다. **husky**와 **lint-staged**를 조합하면 커밋 직전에 자동으로 린트와 포매팅을 강제할 수 있습니다.

```bash
npm install --save-dev husky lint-staged
npx husky init
```

`package.json`에 lint-staged 설정을 추가합니다:

```json
{
  "lint-staged": {
    "*.{js,jsx,ts,tsx}": [
      "eslint --fix",
      "prettier --write"
    ],
    "*.{json,css,md}": [
      "prettier --write"
    ]
  }
}
```

`.husky/pre-commit` 파일:

```sh
npx lint-staged
```

이제 `git commit` 시 스테이징된 파일에만 lint-staged가 실행되어, 문제가 있으면 커밋이 중단됩니다.

---

## npm scripts로 편리하게 실행하기

`package.json`에 편의 스크립트를 추가해두면 CI/CD 파이프라인에서도 활용하기 좋습니다:

```json
{
  "scripts": {
    "lint": "eslint src --ext .js,.jsx,.ts,.tsx",
    "lint:fix": "eslint src --ext .js,.jsx,.ts,.tsx --fix",
    "format": "prettier --write src",
    "format:check": "prettier --check src"
  }
}
```

`format:check`는 Prettier가 파일을 수정하지 않고 포매팅이 맞는지만 확인합니다. CI에서 이 명령어를 실행해 포매팅이 제대로 되지 않은 코드가 머지되는 것을 방지할 수 있습니다.

---

## 도입 순서 요약

1. **Prettier 먼저** 설치하고 팀 합의하에 `.prettierrc` 작성
2. **ESLint** 설치 및 `eslint:recommended` 기반 설정
3. `eslint-config-prettier`로 충돌 제거
4. VS Code 설정으로 저장 시 자동 적용
5. husky + lint-staged로 커밋 전 검사 강제

스타일 논쟁에 쓰던 에너지를 로직과 아키텍처 토론에 쏟아보세요. 도구에 맡길 수 있는 것은 도구에 맡기는 것이 가장 현명한 개발자의 태도입니다.
