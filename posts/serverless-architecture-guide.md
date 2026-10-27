---
title: 서버리스 아키텍처 완전 가이드 — AWS Lambda로 인프라 없이 백엔드 구축하기
date: 2026-10-27
tags: [서버리스, AWS Lambda, 아키텍처, 백엔드, 클라우드]
description: 서버를 직접 관리하지 않고도 확장 가능한 백엔드를 구축하는 서버리스 아키텍처의 핵심 개념과 AWS Lambda 실전 활용법을 소개합니다.
---

클라우드 시대에 접어들면서 "서버리스(Serverless)"라는 단어를 자주 접하게 됩니다. 사실 서버리스라고 해서 서버가 없는 것은 아닙니다. 단지 **개발자가 서버를 직접 프로비저닝·관리·패칭할 필요가 없다**는 뜻입니다. 클라우드 공급자가 인프라를 알아서 처리하고, 개발자는 비즈니스 로직에만 집중할 수 있는 환경이 바로 서버리스입니다.

---

## 서버리스 아키텍처란?

서버리스는 **FaaS(Function as a Service)** 모델에 기반합니다. 함수를 이벤트에 응답하는 독립적인 실행 단위로 배포하고, 해당 함수가 호출될 때만 실행되어 비용이 발생합니다.

대표적인 서버리스 플랫폼:

- **AWS Lambda** — 가장 성숙하고 널리 쓰이는 FaaS 서비스
- **Google Cloud Functions** — GCP 생태계와 긴밀하게 통합
- **Azure Functions** — Microsoft Azure 위에서 동작
- **Cloudflare Workers** — 엣지에서 초저지연으로 실행

---

## 기존 서버 방식 vs. 서버리스

| 항목 | 전통 서버 | 서버리스 |
|------|----------|---------|
| 인프라 관리 | 직접 관리 | 클라우드가 담당 |
| 비용 모델 | 서버 상시 가동 | 실행된 만큼만 과금 |
| 확장성 | 수동 또는 오토스케일링 설정 필요 | 자동 수평 확장 |
| 콜드 스타트 | 없음 | 일부 존재 |
| 상태 관리 | 서버 메모리 활용 가능 | 기본적으로 무상태(Stateless) |

트래픽이 일정하지 않거나 이벤트 기반으로 작동하는 API라면 서버리스가 비용과 운영 부담을 크게 줄여 줍니다.

---

## AWS Lambda 기본 구조

Lambda 함수는 핸들러 함수 하나로 시작합니다.

```javascript
// handler.js
export const handler = async (event, context) => {
  console.log('이벤트 수신:', JSON.stringify(event));

  const name = event.queryStringParameters?.name ?? '세계';

  return {
    statusCode: 200,
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ message: `안녕하세요, ${name}!` }),
  };
};
```

- **event** — API Gateway, S3, DynamoDB Streams 등 트리거에서 전달되는 데이터
- **context** — 함수 실행 환경 정보(남은 실행 시간 등)
- 반환값 — HTTP 응답 형식(API Gateway와 연동 시)

---

## 서버리스 프레임워크로 빠르게 시작하기

**Serverless Framework**나 **AWS SAM**을 쓰면 배포 설정을 코드로 관리할 수 있습니다.

```yaml
# serverless.yml
service: my-api

provider:
  name: aws
  runtime: nodejs20.x
  region: ap-northeast-2

functions:
  hello:
    handler: handler.handler
    events:
      - http:
          path: hello
          method: get
          cors: true
```

배포 명령 한 줄로 Lambda 함수와 API Gateway가 한꺼번에 생성됩니다.

```bash
npx serverless deploy
```

---

## 서버리스 아키텍처의 핵심 패턴

### 1. API 백엔드 패턴

API Gateway → Lambda → DynamoDB 조합이 가장 일반적입니다. CRUD 엔드포인트 각각을 독립 함수로 분리하면 기능별 독립 배포와 세밀한 IAM 권한 설정이 가능합니다.

### 2. 이벤트 기반 처리 패턴

S3에 파일이 업로드되면 Lambda가 자동 실행되어 이미지 리사이징, 문서 변환, 알림 발송 등의 작업을 처리합니다. 비동기 파이프라인을 손쉽게 구성할 수 있는 것이 서버리스의 큰 장점입니다.

### 3. 스케줄 작업 패턴

CloudWatch Events(EventBridge) 크론 규칙으로 정기 배치 작업을 Lambda 함수로 실행합니다. 전용 서버 없이 야간 집계, 데이터 정리, 리포트 생성 작업을 처리할 수 있습니다.

---

## 콜드 스타트 문제와 대응법

서버리스의 대표적인 단점이 **콜드 스타트**입니다. 함수가 오랫동안 호출되지 않으면 컨테이너가 종료되고, 다음 호출 시 재초기화 시간이 발생합니다.

**콜드 스타트를 줄이는 방법:**

1. **Provisioned Concurrency 활성화** — Lambda가 미리 지정된 수의 인스턴스를 웜 상태로 유지
2. **런타임 최적화** — Node.js·Python은 JVM 기반(Java, Kotlin)보다 콜드 스타트가 짧음
3. **번들 크기 최소화** — 불필요한 의존성 제거, 트리 셰이킹 적용
4. **워밍 ping 설정** — CloudWatch Events로 주기적으로 함수 호출(임시방편)

---

## 서버리스가 적합한 경우 vs. 피해야 할 경우

### 적합한 경우
- 이벤트 기반 비동기 처리(파일 업로드, 메시지 큐 처리)
- 트래픽이 불규칙한 API
- 단기 배치 작업
- 사이드 프로젝트·스타트업(운영 비용 절감)

### 피해야 할 경우
- 지속적인 소켓 연결이 필요한 실시간 서비스(WebSocket 주의)
- 15분을 넘는 장시간 실행 작업(Lambda 최대 실행 시간 15분)
- 메모리·CPU를 대량으로 사용하는 고성능 연산

---

## 마무리

서버리스는 인프라 운영 부담 없이 빠르게 백엔드를 구축할 수 있는 강력한 방식입니다. 특히 **이벤트 기반 처리**와 **비정기 트래픽 대응**에 탁월합니다. 처음에는 AWS Lambda + API Gateway + DynamoDB 조합으로 작은 프로젝트부터 시작해 보세요. 서버를 걱정하지 않고 코드에만 집중하는 경험이 어떤 것인지 직접 느껴볼 수 있을 것입니다.
