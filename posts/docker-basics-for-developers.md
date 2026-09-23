---
title: Docker 기초 완전 정복 — 개발자를 위한 컨테이너 입문
date: 2026-09-23
tags: [Docker, 컨테이너, DevOps, 웹 개발, 배포]
description: Docker의 핵심 개념부터 Dockerfile 작성, 기본 명령어까지 실무에 바로 적용할 수 있는 컨테이너 입문 가이드를 소개합니다.
---

"내 컴퓨터에선 잘 되는데요." 개발자라면 한 번쯤 들어봤거나 직접 해본 말일 겁니다. 환경 차이로 인한 버그는 협업과 배포 과정에서 끊임없이 발생합니다. Docker는 바로 이 문제를 해결하기 위해 탄생한 도구입니다.

## Docker란 무엇인가?

Docker는 애플리케이션을 **컨테이너(Container)**라는 격리된 실행 환경에 패키징하고 배포하는 플랫폼입니다. 컨테이너는 애플리케이션과 그 실행에 필요한 모든 것(라이브러리, 설정, 의존성)을 하나로 묶어, 어느 환경에서든 동일하게 동작하도록 보장합니다.

가상 머신(VM)과 자주 비교되는데, 핵심 차이는 다음과 같습니다.

| 항목 | 가상 머신(VM) | 컨테이너 |
|---|---|---|
| 운영체제 | 각자 별도 OS 필요 | 호스트 OS 커널 공유 |
| 시작 시간 | 수십 초 ~ 수 분 | 수 초 이내 |
| 용량 | GB 단위 | MB 단위 |
| 격리 수준 | 강함 | 적절함 |

컨테이너는 가볍고 빠르기 때문에 개발, 테스트, 배포 전 과정에서 폭넓게 활용됩니다.

## 핵심 개념 정리

Docker를 처음 배울 때 헷갈리는 용어들을 먼저 정리합니다.

### 이미지(Image)

이미지는 컨테이너를 만들기 위한 **읽기 전용 템플릿**입니다. 운영체제, 런타임, 라이브러리, 애플리케이션 코드가 레이어 구조로 쌓여 있습니다. 비유하자면, 이미지는 붕어빵 틀이고 컨테이너는 그 틀로 찍어낸 붕어빵입니다.

### 컨테이너(Container)

이미지를 실행한 **실제 인스턴스**입니다. 이미지 하나로 여러 컨테이너를 동시에 실행할 수 있으며, 각 컨테이너는 독립적으로 동작합니다.

### Dockerfile

이미지를 만드는 레시피 파일입니다. 어떤 베이스 이미지를 쓸지, 어떤 명령어를 실행할지, 어떤 파일을 복사할지 등을 단계적으로 정의합니다.

### 레지스트리(Registry)

이미지를 저장하고 공유하는 저장소입니다. 가장 유명한 퍼블릭 레지스트리는 [Docker Hub](https://hub.docker.com)이며, AWS ECR, GitHub Container Registry 같은 프라이빗 레지스트리도 많이 사용합니다.

## 기본 명령어 익히기

실무에서 가장 자주 쓰는 Docker 명령어를 모아봤습니다.

```bash
# 이미지 내려받기
docker pull node:20-alpine

# 컨테이너 실행 (백그라운드, 포트 매핑)
docker run -d -p 3000:3000 --name my-app node:20-alpine

# 실행 중인 컨테이너 목록
docker ps

# 모든 컨테이너 목록 (중지된 것 포함)
docker ps -a

# 컨테이너 로그 확인
docker logs my-app

# 컨테이너 내부 접속
docker exec -it my-app sh

# 컨테이너 중지 및 삭제
docker stop my-app
docker rm my-app

# 이미지 목록
docker images

# 이미지 삭제
docker rmi node:20-alpine
```

## Dockerfile 작성하기

간단한 Node.js 앱을 컨테이너화하는 Dockerfile 예시입니다.

```dockerfile
# 베이스 이미지 선택 (alpine은 경량 리눅스)
FROM node:20-alpine

# 작업 디렉토리 설정
WORKDIR /app

# 의존성 파일 먼저 복사 (캐시 활용)
COPY package*.json ./

# 의존성 설치
RUN npm ci --only=production

# 소스 코드 복사
COPY . .

# 앱이 사용할 포트 명시
EXPOSE 3000

# 컨테이너 시작 시 실행할 명령어
CMD ["node", "server.js"]
```

`package.json`을 소스 코드보다 먼저 복사하는 이유는 **레이어 캐싱** 때문입니다. 의존성이 변경되지 않았다면 `npm ci` 레이어를 재사용해 빌드 속도를 크게 높일 수 있습니다.

이미지를 빌드하고 실행하는 방법입니다.

```bash
# 이미지 빌드 (-t로 태그 지정)
docker build -t my-node-app:1.0 .

# 빌드한 이미지로 컨테이너 실행
docker run -d -p 3000:3000 my-node-app:1.0
```

## .dockerignore 설정

Git의 `.gitignore`처럼, Docker 빌드 컨텍스트에서 제외할 파일을 지정합니다. 이미지 크기를 줄이고 보안을 강화하는 데 필수입니다.

```
node_modules
.git
*.log
.env
dist
coverage
```

## Docker Compose로 여러 서비스 관리하기

실제 프로젝트는 앱 서버, 데이터베이스, 캐시 서버 등 여러 컨테이너가 함께 동작합니다. `docker-compose.yml`로 이를 한 번에 정의하고 관리할 수 있습니다.

```yaml
version: "3.9"

services:
  app:
    build: .
    ports:
      - "3000:3000"
    environment:
      - DATABASE_URL=postgres://user:pass@db:5432/mydb
    depends_on:
      - db

  db:
    image: postgres:15-alpine
    environment:
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=pass
      - POSTGRES_DB=mydb
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

```bash
# 전체 서비스 시작
docker compose up -d

# 로그 확인
docker compose logs -f

# 전체 서비스 중지 및 정리
docker compose down
```

## 실무 활용 팁

### 멀티 스테이지 빌드로 이미지 경량화

프로덕션 이미지에는 빌드 도구가 필요 없습니다. 멀티 스테이지 빌드를 사용하면 최종 이미지 크기를 크게 줄일 수 있습니다.

```dockerfile
# 빌드 스테이지
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# 프로덕션 스테이지
FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/package*.json ./
RUN npm ci --only=production
CMD ["node", "dist/server.js"]
```

### non-root 사용자로 실행하기

보안상 컨테이너를 root로 실행하는 것은 위험합니다. Dockerfile에 사용자를 지정하세요.

```dockerfile
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
```

## 마무리

Docker는 처음엔 낯설게 느껴지지만, 익히고 나면 개발 환경의 일관성과 배포 편의성이 눈에 띄게 향상됩니다. 오늘 소개한 기본 명령어와 Dockerfile 작성법을 먼저 익히고, 이후 Docker Compose와 CI/CD 파이프라인 통합으로 단계적으로 넓혀가는 것을 추천합니다.

컨테이너 기술은 현대 개발 환경의 핵심입니다. 지금 바로 로컬 환경에 Docker를 설치하고 간단한 앱을 컨테이너화해 보세요.
