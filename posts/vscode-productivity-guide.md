---
title: VSCode 완전 활용 가이드 — 생산성을 높이는 확장 프로그램과 설정 꿀팁
date: 2026-10-26
tags: [VSCode, 개발 환경, 생산성, 에디터, IDE]
description: VSCode의 핵심 확장 프로그램과 숨겨진 설정으로 코딩 속도와 편의성을 극대화하는 실전 가이드입니다.
---

## Visual Studio Code가 선택받는 이유

Visual Studio Code(이하 VSCode)는 현재 전 세계에서 가장 많이 사용되는 코드 에디터입니다. 가볍고 빠른 성능, 강력한 확장 생태계, 무료라는 장점 덕분에 웹 개발자부터 데이터 사이언티스트까지 광범위하게 사용됩니다.

하지만 많은 개발자들이 기본 설정 그대로 사용하면서 VSCode의 강력한 기능을 놓치고 있습니다. 오늘은 VSCode를 최대한 활용해 코딩 생산성을 극대화하는 방법을 알아보겠습니다.

## 반드시 설치해야 할 확장 프로그램

### 코드 품질 & 포매팅

**Prettier — Code formatter**
코드 스타일을 자동으로 일관성 있게 정리해 주는 필수 확장 프로그램입니다. 팀 프로젝트에서는 `.prettierrc` 파일로 규칙을 공유하면 코드 스타일 논쟁을 없앨 수 있습니다.

**ESLint**
JavaScript/TypeScript 코드의 문법 오류와 잠재적 버그를 실시간으로 감지합니다. 저장할 때 자동으로 수정되도록 설정하면 더욱 효율적입니다.

**Error Lens**
에러와 경고를 인라인으로 직접 에디터에 표시해 줍니다. 별도의 Problems 패널을 열지 않아도 코드 옆에서 바로 문제를 확인할 수 있어 디버깅 시간이 크게 줄어듭니다.

### 생산성 향상

**GitHub Copilot**
AI 기반 코드 자동 완성 도구로, 주석만 작성해도 함수 구현을 제안해 줍니다. 반복적인 보일러플레이트 코드 작성 시간을 크게 줄여 줍니다.

**GitLens**
각 코드 라인에 git blame 정보를 오버레이로 표시하고, 커밋 히스토리, 변경 내역 비교 등을 에디터 안에서 직접 확인할 수 있습니다.

**Todo Tree**
코드 내의 `TODO`, `FIXME`, `HACK` 등의 주석을 한눈에 모아볼 수 있는 패널을 제공합니다. 작업 목록 관리에 탁월합니다.

**Path Intellisense**
파일 경로를 자동 완성해 줍니다. `import` 구문이나 파일 경로를 입력할 때 디렉토리 구조를 탐색하며 자동 완성해 줘 실수를 방지합니다.

### 외관 & 편의성

**One Dark Pro** 또는 **GitHub Theme**
눈에 피로를 줄이고 코드 가독성을 높이는 색상 테마입니다. 오랜 시간 코딩할수록 좋은 테마의 중요성이 커집니다.

**Material Icon Theme**
파일 탐색기의 아이콘을 파일 유형별로 시각적으로 구분해 주어 파일을 더 빠르게 찾을 수 있습니다.

## 꼭 알아야 할 핵심 단축키

단축키를 익히면 마우스 없이 작업하는 시간이 늘어나 생산성이 크게 향상됩니다.

| 기능 | Windows/Linux | macOS |
|------|-------------|-------|
| 커맨드 팔레트 | `Ctrl+Shift+P` | `Cmd+Shift+P` |
| 빠른 파일 열기 | `Ctrl+P` | `Cmd+P` |
| 현재 줄 삭제 | `Ctrl+Shift+K` | `Cmd+Shift+K` |
| 줄 이동 (위/아래) | `Alt+↑/↓` | `Option+↑/↓` |
| 같은 단어 선택 추가 | `Ctrl+D` | `Cmd+D` |
| 모든 같은 단어 선택 | `Ctrl+Shift+L` | `Cmd+Shift+L` |
| 멀티 커서 (클릭) | `Alt+Click` | `Option+Click` |
| 통합 터미널 열기 | `` Ctrl+` `` | `` Cmd+` `` |

## settings.json 최적화

VSCode의 설정은 GUI보다 `settings.json`을 직접 편집하면 더 세밀하게 제어할 수 있습니다. `Ctrl+Shift+P` → "Open User Settings (JSON)"으로 접근합니다.

```json
{
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",

  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": "explicit"
  },

  "editor.fontFamily": "Fira Code, Consolas, monospace",
  "editor.fontLigatures": true,
  "editor.fontSize": 14,
  "editor.lineHeight": 1.6,

  "explorer.fileNesting.enabled": true,
  "explorer.fileNesting.patterns": {
    "*.ts": "${capture}.js, ${capture}.d.ts",
    "package.json": "package-lock.json, yarn.lock, pnpm-lock.yaml"
  },

  "files.autoSave": "onFocusChange",

  "editor.bracketPairColorization.enabled": true,

  "editor.minimap.enabled": false,

  "editor.stickyScroll.enabled": true
}
```

`editor.stickyScroll.enabled`를 켜면 스크롤해도 현재 위치한 함수나 클래스 이름이 상단에 고정되어 코드 구조 파악이 훨씬 쉬워집니다.

## 커스텀 스니펫(Snippet) 활용

반복적으로 입력하는 코드 패턴은 스니펫으로 등록해 두면 큰 시간을 아낄 수 있습니다. `Ctrl+Shift+P` → "Configure User Snippets"에서 언어별로 설정합니다.

```json
// javascript.json
{
  "Console Log": {
    "prefix": "cl",
    "body": ["console.log('$1:', $1);"],
    "description": "console.log with label"
  },
  "Arrow Function": {
    "prefix": "af",
    "body": [
      "const $1 = ($2) => {",
      "  $3",
      "};"
    ],
    "description": "Arrow function"
  },
  "React Functional Component": {
    "prefix": "rfc",
    "body": [
      "interface ${1:Component}Props {}",
      "",
      "export default function ${1:Component}({}: ${1:Component}Props) {",
      "  return (",
      "    <div>$2</div>",
      "  );",
      "}"
    ],
    "description": "React functional component with TypeScript"
  }
}
```

`prefix`에 짧은 약어를 설정해 두면 몇 글자만 입력해도 전체 코드 블록이 자동 완성됩니다.

## 멀티 커서로 반복 작업 자동화

VSCode의 멀티 커서 기능은 반복적인 텍스트 편집 작업에서 강력한 효율을 발휘합니다.

- **같은 단어 선택**: `Ctrl+D`를 반복해 같은 단어를 순차적으로 선택하거나, `Ctrl+Shift+L`로 한 번에 모두 선택
- **세로 멀티 커서**: `Ctrl+Alt+↑/↓`로 위아래로 커서 추가
- **선택 영역 확장**: `Alt+Shift+→`로 선택 범위를 점진적으로 넓히기
- **직사각형 선택**: `Shift+Alt`를 누른 채 드래그하면 열(column) 단위로 선택

예를 들어 여러 변수 이름을 동시에 바꾸거나, 여러 줄에 동일한 접두사를 추가할 때 큰 효과를 발휘합니다.

## 작업 공간(Workspace) 분리 전략

프로젝트마다 별도의 Workspace를 설정하면 확장 프로그램 활성화 여부, 설정값, 디버그 설정을 개별 관리할 수 있습니다.

```json
// .vscode/settings.json (프로젝트별 설정)
{
  "editor.tabSize": 2,
  "typescript.preferences.importModuleSpecifier": "relative",
  "search.exclude": {
    "**/node_modules": true,
    "**/dist": true,
    "**/.git": true
  }
}
```

팀 레포지터리에 `.vscode/settings.json`과 `.vscode/extensions.json`을 포함시키면 팀원 모두가 동일한 환경에서 개발할 수 있습니다.

```json
// .vscode/extensions.json
{
  "recommendations": [
    "esbenp.prettier-vscode",
    "dbaeumer.vscode-eslint",
    "usernamehw.errorlens",
    "eamodio.gitlens"
  ]
}
```

이 파일을 커밋해 두면 동료가 해당 프로젝트를 열 때 추천 확장 프로그램 설치 알림이 뜨게 됩니다.

## 내장 Git 기능 200% 활용하기

별도의 GUI 클라이언트 없이도 VSCode의 내장 Git 기능만으로 대부분의 작업을 처리할 수 있습니다.

- **Source Control 패널** (`Ctrl+Shift+G`): 변경 파일 목록, 스테이징, 커밋을 GUI로 관리
- **인라인 diff 보기**: 파일 탐색기에서 변경된 파일을 클릭하면 이전 버전과 나란히 비교
- **Timeline 뷰**: 파일 탐색기 하단 Timeline 패널에서 파일의 커밋 히스토리 확인
- **Merge Editor**: 충돌 발생 시 3-way merge 에디터로 직관적인 해결

## 마치며

VSCode는 그 자체로도 강력하지만, 본인의 작업 흐름에 맞게 설정과 확장 프로그램을 최적화했을 때 진가를 발휘합니다. 오늘 소개한 확장 프로그램과 설정을 하나씩 적용해 보면서 자신만의 최적화된 개발 환경을 만들어 보세요.

처음에는 설정하는 데 시간이 걸리더라도, 매일 쌓이는 시간 절약과 편의성은 그 투자를 충분히 보상해 줍니다. 특히 팀 프로젝트라면 `.vscode` 폴더를 공유해 팀 전체의 개발 환경을 통일하는 것부터 시작해 보길 권장합니다.
