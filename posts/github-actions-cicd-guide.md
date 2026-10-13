---
title: GitHub Actions로 CI/CD 파이프라인 구축하기 — 자동화로 배포를 한 단계 업그레이드
date: 2026-10-09
tags: [GitHub Actions, CI/CD, DevOps, 자동화, 배포]
description: GitHub Actions를 활용해 테스트·빌드·배포를 자동화하는 CI/CD 파이프라인을 단계별로 구축하는 방법을 소개합니다.
---

코드를 작성하고 서버에 배포하는 과정을 매번 수동으로 반복하고 있다면, 이제 자동화를 도입할 때입니다. GitHub Actions는 GitHub 저장소에 내장된 CI/CD 플랫폼으로, 별도의 외부 서비스 없이도 강력한 자동화 파이프라인을 구축할 수 있습니다.

이 글에서는 GitHub Actions의 기본 개념부터 실전 파이프라인 구성까지 단계별로 살펴봅니다.

## GitHub Actions란?

GitHub Actions는 저장소에서 발생하는 이벤트(push, PR 생성, 스케줄 등)에 반응해 자동으로 작업을 실행하는 자동화 플랫폼입니다. 2019년 정식 출시 이후 수많은 팀이 Jenkins, CircleCI 등의 외부 CI/CD 도구를 대체하는 수단으로 채택하고 있습니다.

### 핵심 개념 한눈에 보기

- **Workflow**: 자동화 파이프라인 전체 단위. `.github/workflows/` 폴더에 YAML 파일로 정의합니다.
- **Event**: Workflow를 실행시키는 트리거. `push`, `pull_request`, `schedule` 등이 있습니다.
- **Job**: Workflow 안에서 독립적으로 실행되는 작업 단위. 병렬 또는 순차 실행이 가능합니다.
- **Step**: Job 안에서 순서대로 실행되는 개별 명령. 셸 명령이나 Action을 사용합니다.
- **Action**: 재사용 가능한 작업 단위. GitHub Marketplace에서 수천 개의 공개 Action을 활용할 수 있습니다.

## 첫 번째 Workflow 만들기

가장 기본적인 예시로, main 브랜치에 push가 발생할 때마다 Node.js 테스트를 자동으로 실행하는 Workflow를 만들어 보겠습니다.

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - name: 코드 체크아웃
        uses: actions/checkout@v4

      - name: Node.js 설정
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: 의존성 설치
        run: npm ci

      - name: 린트 실행
        run: npm run lint

      - name: 테스트 실행
        run: npm test
```

이 파일을 저장소에 푸시하면 즉시 Workflow가 활성화됩니다. PR을 열거나 main 브랜치에 푸시할 때마다 테스트가 자동으로 실행됩니다.

## 실전 CI/CD 파이프라인 구성

### 1단계: 테스트 자동화 (CI)

테스트 Job에서는 코드 품질을 검증하는 모든 작업을 수행합니다.

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [18, 20, 22]  # 여러 버전에서 동시 테스트

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
      - run: npm ci
      - run: npm run build
      - run: npm test -- --coverage
      - name: 커버리지 업로드
        uses: codecov/codecov-action@v4
```

`matrix` 전략을 사용하면 여러 환경에서 동시에 테스트를 실행할 수 있어 호환성 문제를 조기에 발견할 수 있습니다.

### 2단계: 빌드 및 배포 자동화 (CD)

테스트가 통과된 후에만 배포가 실행되도록 `needs` 키워드로 Job 간 의존성을 설정합니다.

```yaml
  deploy:
    runs-on: ubuntu-latest
    needs: test  # test Job이 성공해야 실행
    if: github.ref == 'refs/heads/main'  # main 브랜치에서만 배포

    steps:
      - uses: actions/checkout@v4

      - name: Docker 이미지 빌드 및 푸시
        uses: docker/build-push-action@v5
        with:
          push: true
          tags: myapp:latest

      - name: 서버 배포
        run: |
          ssh user@server "docker pull myapp:latest && docker-compose up -d"
        env:
          SSH_KEY: ${{ secrets.DEPLOY_SSH_KEY }}
```

### 3단계: 환경 분리

실무에서는 개발(dev), 스테이징(staging), 프로덕션(production) 환경을 분리하여 관리하는 것이 중요합니다.

```yaml
on:
  push:
    branches:
      - develop   # dev 환경 배포
      - staging   # staging 환경 배포
      - main      # production 배포

jobs:
  deploy:
    environment: ${{ github.ref_name }}  # 브랜치명을 환경명으로 사용
```

GitHub의 **Environments** 기능을 활용하면 환경별로 보호 규칙(승인 요청, 대기 시간 등)과 시크릿을 별도로 관리할 수 있습니다.

## 유용한 실전 팁

### 캐시 활용으로 속도 높이기

의존성 캐시를 적극적으로 활용하면 실행 시간을 크게 단축할 수 있습니다.

```yaml
- uses: actions/cache@v4
  with:
    path: ~/.npm
    key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
```

### Secrets로 민감 정보 관리

API 키, 배포 토큰 등 민감한 정보는 절대 코드에 직접 작성하지 말고, 저장소의 **Settings → Secrets and variables → Actions**에 등록한 후 `${{ secrets.SECRET_NAME }}` 형식으로 참조합니다.

### Workflow 재사용

여러 저장소에서 동일한 Workflow를 반복 작성하는 대신, Reusable Workflow로 한 번 정의하고 여러 곳에서 호출할 수 있습니다.

```yaml
# 재사용 가능한 Workflow 호출
jobs:
  call-workflow:
    uses: my-org/.github/.github/workflows/deploy.yml@main
    with:
      environment: production
    secrets: inherit
```

## 마치며

GitHub Actions는 진입 장벽이 낮으면서도 복잡한 파이프라인을 구성할 수 있는 강력한 도구입니다. 처음에는 간단한 테스트 자동화부터 시작해, 점차 빌드·배포·알림까지 파이프라인을 확장해 나가는 것이 좋습니다.

자동화된 CI/CD 파이프라인이 갖춰지면 코드 리뷰와 기능 개발에 더 집중할 수 있고, 릴리즈 주기를 단축하면서도 품질은 높게 유지할 수 있습니다. 반복적인 수동 배포 작업에서 해방되고 싶다면, 지금 바로 첫 번째 Workflow 파일을 만들어 보세요.
