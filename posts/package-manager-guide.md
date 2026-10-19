---
title: 패키지 매니저 완전 가이드 — npm, Yarn, pnpm 비교와 팀에 맞는 선택 기준
date: 2026-10-19
tags: [패키지 매니저, npm, Yarn, pnpm, 개발 환경, Node.js]
description: npm, Yarn, pnpm 세 가지 패키지 매니저의 핵심 차이와 장단점을 비교해 프로젝트에 맞는 최적의 선택 기준을 제시합니다.
---

## 왜 패키지 매니저가 중요한가

현대 자바스크립트 프로젝트는 수백 개의 외부 패키지에 의존합니다. 패키지 매니저는 이 의존성을 설치·관리·업데이트하는 핵심 도구로, 개발 환경의 안정성과 팀 협업의 일관성에 직접적인 영향을 미칩니다.

패키지 매니저를 잘못 선택하거나 제대로 이해하지 못하면 다음과 같은 문제가 생깁니다.

- **의존성 충돌**로 "내 컴퓨터에서는 되는데..." 문제가 반복됩니다.
- **느린 설치 속도**로 CI/CD 파이프라인이 병목이 됩니다.
- **디스크 공간 낭비**로 `node_modules` 폴더가 수 GB를 차지하게 됩니다.

이 글에서는 npm, Yarn, pnpm 세 가지 주요 패키지 매니저를 깊이 비교하고, 상황에 맞는 선택 기준을 제시합니다.

---

## npm — Node.js의 기본 패키지 매니저

npm(Node Package Manager)은 Node.js와 함께 설치되는 기본 패키지 매니저입니다. 가장 오랜 역사를 가지며 생태계가 가장 넓습니다.

### 핵심 명령어

```bash
# 패키지 설치
npm install react

# 개발 의존성 설치
npm install --save-dev typescript

# 전역 설치
npm install -g nodemon

# package.json 기반 의존성 전체 설치
npm install

# 스크립트 실행
npm run build
```

### npm의 장단점

**장점**

- Node.js에 기본 내장 — 별도 설치 불필요
- 가장 큰 커뮤니티와 레퍼런스 자료
- 워크스페이스(모노레포) 지원 (npm v7+)

**단점**

- Yarn, pnpm에 비해 상대적으로 느린 설치 속도
- `node_modules` 구조가 비효율적 (동일 패키지 중복 저장)
- 과거 보안 취약점 이력 (현재는 많이 개선됨)

### lock 파일의 중요성

`package-lock.json`은 의존성 트리의 정확한 버전을 고정합니다. 반드시 git에 커밋해야 팀원들이 동일한 환경에서 작업할 수 있습니다.

```bash
# lock 파일을 무시하고 최신 버전으로 설치 (권장하지 않음)
npm install --legacy-peer-deps

# lock 파일 기반 정확한 버전으로 설치 (CI 환경 권장)
npm ci
```

---

## Yarn — 속도와 안정성을 앞세운 도전자

Yarn은 2016년 Facebook(현 Meta)이 npm의 속도와 안정성 문제를 해결하기 위해 출시했습니다. 현재는 Yarn Classic(v1)과 Yarn Berry(v2+) 두 가지 버전이 공존합니다.

### Yarn의 특징적인 기능

```bash
# 설치
npm install -g yarn

# 패키지 추가
yarn add react

# 개발 의존성 추가
yarn add --dev typescript

# 전체 설치 (병렬 처리로 빠름)
yarn install

# 스크립트 실행
yarn build
```

### Yarn Berry의 Plug'n'Play (PnP)

Yarn Berry의 가장 혁신적인 기능은 **Plug'n'Play** 방식입니다. 기존의 `node_modules` 폴더를 없애고, 의존성을 압축 파일(`.yarn/cache`)로 관리합니다.

```
.yarn/
  cache/       # 패키지 압축 파일 (zip)
  releases/    # Yarn 바이너리
.pnp.cjs       # 의존성 해석 맵
```

**PnP의 장점:**
- `node_modules` 폴더 불필요 → 디스크 공간 절약
- 의존성 설치 후 거의 즉시 사용 가능 (`node_modules` 생성 과정 없음)
- 더 엄격한 의존성 격리로 유령 의존성 방지

**PnP의 단점:**
- 일부 오래된 패키지가 PnP와 호환되지 않음
- VSCode에서 ZipFS 확장 등 추가 IDE 설정 필요

---

## pnpm — 공유 스토어로 공간과 속도를 혁신

pnpm은 "performant npm"의 약자로, 혁신적인 **콘텐츠 주소 기반 스토어(Content-addressable store)** 방식을 사용합니다.

### pnpm의 동작 원리

```
전역 스토어: ~/.pnpm-store/
    react@18.2.0/
    typescript@5.0.4/
    ...

프로젝트 A/node_modules/
    react → ~/.pnpm-store/react@18.2.0  (하드 링크)

프로젝트 B/node_modules/
    react → ~/.pnpm-store/react@18.2.0  (같은 파일 공유!)
```

동일한 패키지를 여러 프로젝트에서 사용해도 디스크에는 단 한 번만 저장됩니다. 여러 프로젝트를 동시에 운영하는 개발자라면 수 GB의 공간을 절약할 수 있습니다.

### pnpm 핵심 명령어

```bash
# 설치
npm install -g pnpm

# 패키지 추가
pnpm add react

# 개발 의존성
pnpm add -D typescript

# 전체 설치
pnpm install

# 스크립트 실행
pnpm build

# 모노레포에서 모든 패키지 빌드
pnpm -r build
```

### pnpm의 장단점

**장점**

- 대부분의 벤치마크에서 가장 빠른 설치 속도
- 디스크 공간 대폭 절약 (하드 링크 공유)
- 엄격한 의존성 격리로 유령 의존성 문제 방지
- 우수한 모노레포(워크스페이스) 지원

**단점**

- 공유 스토어 방식이 낯설어 처음에 진입 장벽이 있음
- 일부 레거시 패키지와의 호환성 문제 발생 가능
- npm/Yarn보다 커뮤니티가 작음

---

## 성능 비교

주요 패키지 매니저의 특성을 한눈에 비교합니다.

| 항목 | npm | Yarn Classic | pnpm |
|------|:---:|:-----------:|:----:|
| 설치 속도 | 보통 | 빠름 | 가장 빠름 |
| 디스크 효율 | 낮음 | 보통 | 높음 |
| 캐시 활용 | 보통 | 좋음 | 뛰어남 |
| 모노레포 지원 | 가능 | 좋음 | 뛰어남 |
| 의존성 격리 | 느슨함 | 느슨함 | 엄격함 |
| 러닝 커브 | 낮음 | 낮음 | 중간 |
| 추가 설치 필요 | 불필요 | 필요 | 필요 |

---

## 어떤 패키지 매니저를 선택해야 할까?

### 상황별 추천

**새 프로젝트를 시작하는 경우**  
→ **pnpm** 권장. 속도와 디스크 효율이 모두 뛰어나며, 엄격한 의존성 격리로 예측 가능한 환경을 만들어줍니다.

**기존 npm 프로젝트를 유지하는 경우**  
→ **npm 유지** 또는 pnpm으로 점진적 마이그레이션. 큰 문제가 없다면 굳이 교체할 필요는 없습니다.

**모노레포를 운영하는 경우**  
→ **pnpm 워크스페이스** 또는 **Yarn Berry**. 두 도구 모두 모노레포에 최적화되어 있습니다.

**팀의 진입 장벽을 낮추고 싶은 경우**  
→ **npm**. 별도 설치 없이 사용 가능하고 레퍼런스가 가장 풍부합니다.

### 선택 체크리스트

- [ ] 팀원들의 Node.js/패키지 매니저 버전이 통일되어 있는가?
- [ ] 프로젝트에 모노레포 구조가 필요한가?
- [ ] CI/CD 파이프라인의 설치 속도가 중요한가?
- [ ] 레거시 패키지 의존성이 많은가?
- [ ] 팀이 새로운 도구 학습에 충분한 여유가 있는가?

---

## npm에서 pnpm으로 마이그레이션하기

실무에서 가장 많이 묻는 마이그레이션 방법입니다. 다음 순서를 따르세요.

```bash
# 1. pnpm 전역 설치
npm install -g pnpm

# 2. 기존 lock 파일 제거
rm package-lock.json   # 또는 yarn.lock

# 3. node_modules 삭제
rm -rf node_modules

# 4. pnpm으로 의존성 설치
pnpm install

# 5. pnpm-lock.yaml 생성 확인 후 커밋
git add pnpm-lock.yaml
git commit -m "chore: migrate to pnpm"
```

> **주의:** CI/CD 환경(GitHub Actions 등)에서도 pnpm을 설치하도록 워크플로우 파일을 업데이트해야 합니다.

```yaml
# GitHub Actions에서 pnpm 사용 예시
- uses: pnpm/action-setup@v4
  with:
    version: 9
- uses: actions/setup-node@v4
  with:
    node-version: '20'
    cache: 'pnpm'
- run: pnpm install --frozen-lockfile
```

---

## 정리

패키지 매니저는 개발 환경의 기반입니다. 프로젝트의 규모, 팀의 익숙함, 성능 요구사항을 종합적으로 고려해 선택하세요.

- **단순함이 우선**이라면 → npm
- **빠른 설치가 필요**하다면 → Yarn Classic 또는 pnpm
- **최고 성능 + 모노레포**라면 → pnpm

어떤 도구를 선택하든, **lock 파일을 반드시 커밋**하는 습관이 팀 협업의 일관성을 지키는 가장 중요한 원칙입니다. 패키지 버전이 고정되지 않으면 팀원마다 다른 환경에서 개발하게 되고, 예측 불가능한 버그의 온상이 됩니다.
