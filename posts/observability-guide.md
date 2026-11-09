---
title: Observability 완전 가이드 — 로그, 메트릭, 트레이싱으로 시스템을 투명하게 운영하기
date: 2026-11-09
tags: [Observability, 모니터링, 로그, 메트릭, DevOps]
description: 현대 분산 시스템을 투명하게 운영하기 위한 세 가지 핵심 기둥 — 로그, 메트릭, 분산 트레이싱의 개념과 실전 활용법을 소개합니다.
---

## Observability란 무엇인가?

소프트웨어 시스템이 복잡해질수록 "왜 이렇게 동작하는가?"를 이해하기가 어려워집니다. 단순한 모니터링(Monitoring)이 "지금 잘 돌아가고 있는가?"를 확인하는 것이라면, **Observability(관찰 가능성)** 는 "시스템 내부에서 무슨 일이 일어나고 있는가?"를 외부에서 파악할 수 있는 능력을 뜻합니다.

제어 이론에서 유래한 이 개념은 세 가지 핵심 기둥으로 구성됩니다.

- **로그(Logs)**: 특정 시점에 발생한 이벤트의 텍스트 기록
- **메트릭(Metrics)**: 시간에 따른 수치 데이터의 집계
- **분산 트레이싱(Distributed Tracing)**: 여러 서비스를 거치는 요청의 흐름 추적

이 세 가지를 조합하면 장애 원인을 빠르게 파악하고 성능 병목을 제거할 수 있습니다.

---

## 로그(Logs): 이벤트를 기록하다

로그는 가장 오래되고 직관적인 관찰 수단입니다. 그러나 단순히 `console.log`를 남발하는 것으로는 부족합니다.

### 구조화된 로그(Structured Logging)

JSON 형태로 로그를 남기면 검색과 집계가 쉬워집니다.

```json
{
  "timestamp": "2026-11-09T10:23:45Z",
  "level": "error",
  "service": "payment-api",
  "traceId": "abc123",
  "userId": "user_789",
  "message": "결제 처리 실패",
  "error": "insufficient_funds"
}
```

### 로그 레벨 전략

| 레벨 | 사용 시점 |
|------|-----------|
| `DEBUG` | 개발 중 상세 정보 |
| `INFO` | 주요 비즈니스 이벤트 |
| `WARN` | 잠재적 문제, 복구 가능 |
| `ERROR` | 즉각 대응이 필요한 오류 |
| `FATAL` | 시스템 중단 수준의 오류 |

프로덕션 환경에서는 `INFO` 이상만 수집하되, 문제 발생 시 `DEBUG` 레벨로 일시적으로 낮추는 전략이 효과적입니다.

---

## 메트릭(Metrics): 수치로 건강을 측정하다

메트릭은 시스템의 현재 상태를 숫자로 표현합니다. 대표적인 메트릭 유형은 네 가지입니다.

### 핵심 메트릭 유형

```
Counter   — 단조증가 값 (요청 수, 오류 횟수)
Gauge     — 현재 상태 값 (메모리 사용량, 활성 연결 수)
Histogram — 값의 분포 (응답 시간 p50/p95/p99)
Summary   — 클라이언트 측 분위수 계산
```

### USE & RED 방법론

- **USE**: Utilization(사용률), Saturation(포화도), Errors(오류율) — 인프라 메트릭에 적합
- **RED**: Rate(처리율), Errors(오류율), Duration(지연 시간) — 서비스 메트릭에 적합

```js
// Prometheus 스타일 메트릭 예시 (Node.js)
const httpRequestDuration = new Histogram({
  name: 'http_request_duration_seconds',
  help: 'HTTP 요청 처리 시간',
  labelNames: ['method', 'route', 'status_code'],
  buckets: [0.01, 0.05, 0.1, 0.5, 1, 2, 5]
});
```

---

## 분산 트레이싱(Distributed Tracing): 요청의 여정을 추적하다

마이크로서비스 환경에서 단일 요청이 수십 개의 서비스를 거칠 수 있습니다. 분산 트레이싱은 이 요청의 전체 경로를 하나의 타임라인으로 시각화합니다.

### 핵심 개념

- **Trace**: 하나의 요청에 대한 전체 여정
- **Span**: Trace를 구성하는 개별 작업 단위 (서비스 호출, DB 쿼리 등)
- **Context Propagation**: 서비스 간 Trace ID 전달

```js
// OpenTelemetry 예시
const tracer = trace.getTracer('payment-service');

const span = tracer.startSpan('processPayment', {
  attributes: {
    'payment.amount': 15000,
    'payment.currency': 'KRW',
    'user.id': userId
  }
});

try {
  await chargeCard(paymentInfo);
  span.setStatus({ code: SpanStatusCode.OK });
} catch (err) {
  span.recordException(err);
  span.setStatus({ code: SpanStatusCode.ERROR });
} finally {
  span.end();
}
```

---

## 실전 도구 스택

### 오픈소스 조합 (Self-Hosted)

```
로그    → Loki + Promtail
메트릭  → Prometheus + Alertmanager
트레이싱→ Jaeger 또는 Tempo
시각화  → Grafana
```

### 관리형 서비스 (Managed)

| 도구 | 특징 |
|------|------|
| Datadog | 세 가지 기둥 통합, 강력한 APM |
| New Relic | 풀스택 관찰 가능성 |
| AWS CloudWatch | AWS 환경 최적화 |
| Google Cloud Ops | GCP 네이티브 통합 |

---

## SLI, SLO, SLA: 신뢰성의 언어

Observability 데이터를 활용해 신뢰성을 정량화할 수 있습니다.

- **SLI(Service Level Indicator)**: 측정 가능한 신뢰성 지표 (가용성, 응답 시간 등)
- **SLO(Service Level Objective)**: SLI의 목표값 (예: 99.9% 가용성)
- **SLA(Service Level Agreement)**: SLO를 기반으로 한 계약

```
SLI: 성공 응답 비율 = 성공 요청 수 / 전체 요청 수
SLO: 28일 중 99.9% 이상 성공
에러 버짓: 한 달 약 43분의 허용 다운타임
```

에러 버짓(Error Budget)이 소진되기 전에 알림을 받도록 설정하면, 릴리스 속도와 안정성 사이의 균형을 데이터 기반으로 결정할 수 있습니다.

---

## 도입할 때 고려할 점

### 1. 처음부터 너무 많이 수집하지 않기

모든 것을 로깅하고 측정하면 비용이 급격히 증가합니다. 비즈니스 임팩트가 높은 지표부터 시작해 점진적으로 확장하세요.

### 2. 카디널리티(Cardinality) 주의

메트릭 레이블에 사용자 ID처럼 고유값이 많은 필드를 넣으면 메모리와 비용이 폭증합니다. 로그에 담고 메트릭에는 집계된 카테고리만 사용하세요.

### 3. 알림 피로(Alert Fatigue) 방지

알림이 너무 많으면 중요한 신호를 놓칩니다. 알림은 반드시 사람이 조치해야 하는 경우에만 발생하도록 임계값과 조건을 엄격하게 설정하세요.

---

## 마무리

Observability는 단순한 도구 도입이 아니라 **시스템을 투명하게 만드는 문화**입니다. 로그, 메트릭, 트레이싱 세 가지를 유기적으로 연결하면 장애 발생 시 평균 복구 시간(MTTR)을 크게 단축하고, 성능 개선의 근거를 데이터로 제시할 수 있습니다.

지금 당장 모든 것을 완벽하게 갖출 필요는 없습니다. 가장 중요한 서비스 하나에 구조화된 로그와 기본 RED 메트릭을 추가하는 것부터 시작해 보세요. 작은 투명성이 큰 신뢰성으로 이어집니다.
