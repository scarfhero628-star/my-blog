---
title: Kubernetes 입문 가이드 — Docker를 넘어 컨테이너 오케스트레이션으로
date: 2026-10-20
tags: [Kubernetes, 컨테이너, DevOps, 오케스트레이션, Docker]
description: Docker 컨테이너를 자동으로 배포·확장·관리하는 Kubernetes의 핵심 개념과 기본 명령어를 실전 예제와 함께 소개합니다.
---

Docker를 배우고 나면 자연스럽게 다음 질문이 찾아옵니다. "컨테이너가 여러 개가 되면 어떻게 관리하지?" 서비스가 커질수록 수십, 수백 개의 컨테이너를 손으로 관리하는 건 불가능에 가깝습니다. 바로 이 문제를 해결하기 위해 등장한 것이 **Kubernetes(쿠버네티스, K8s)**입니다.

## Kubernetes란 무엇인가?

Kubernetes는 구글이 내부적으로 사용하던 컨테이너 관리 시스템 'Borg'를 오픈소스화한 프로젝트입니다. 2014년 공개 이후 CNCF(Cloud Native Computing Foundation)가 관리하며 사실상 컨테이너 오케스트레이션의 표준이 되었습니다.

Kubernetes가 해결하는 핵심 문제는 다음과 같습니다.

- **자동 배포와 롤백**: 새 버전을 배포하다 문제가 생기면 자동으로 이전 버전으로 복구합니다.
- **자가 치유(Self-healing)**: 컨테이너가 죽으면 자동으로 재시작하고, 의도한 상태를 유지합니다.
- **수평 확장(Horizontal Scaling)**: CPU 사용률 같은 지표에 따라 컨테이너 수를 자동으로 늘리거나 줄입니다.
- **서비스 디스커버리와 로드 밸런싱**: 여러 컨테이너에 트래픽을 자동으로 분산합니다.
- **설정과 시크릿 관리**: 데이터베이스 비밀번호 같은 민감 정보를 안전하게 주입합니다.

## 핵심 개념 이해하기

### Pod

Kubernetes에서 가장 작은 배포 단위입니다. 하나 이상의 컨테이너 묶음으로, 같은 Pod 안의 컨테이너들은 네트워크와 스토리지를 공유합니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
  - name: app
    image: nginx:latest
    ports:
    - containerPort: 80
```

### Deployment

Pod를 직접 만드는 대신 Deployment를 사용합니다. 몇 개의 Pod를 항상 실행할지, 어떤 이미지를 사용할지 선언하면 Kubernetes가 그 상태를 유지합니다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3          # Pod 3개를 항상 유지
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: app
        image: my-app:1.0.0
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
```

### Service

Pod는 재시작될 때마다 IP가 바뀝니다. Service는 변하지 않는 고정 엔드포인트를 제공해 Pod들 앞에 로드 밸런서 역할을 합니다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-service
spec:
  selector:
    app: my-app
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
  type: ClusterIP    # ClusterIP | NodePort | LoadBalancer
```

### Namespace

클러스터 안에서 리소스를 논리적으로 격리하는 단위입니다. 개발·스테이징·프로덕션 환경을 같은 클러스터에서 분리할 때 활용합니다.

## 자주 쓰는 kubectl 명령어

`kubectl`은 Kubernetes 클러스터와 소통하는 CLI 도구입니다.

```bash
# 클러스터 상태 확인
kubectl cluster-info

# 리소스 목록 조회
kubectl get pods
kubectl get deployments
kubectl get services

# 특정 네임스페이스 조회
kubectl get pods -n production

# 리소스 상세 정보
kubectl describe pod my-app-xxxx

# 로그 확인
kubectl logs my-app-xxxx
kubectl logs -f my-app-xxxx   # 실시간 스트리밍

# 컨테이너 접속
kubectl exec -it my-app-xxxx -- /bin/sh

# Deployment 이미지 업데이트 (롤링 업데이트)
kubectl set image deployment/my-app app=my-app:2.0.0

# 롤백
kubectl rollout undo deployment/my-app

# YAML 파일로 리소스 생성/업데이트
kubectl apply -f deployment.yaml

# 리소스 삭제
kubectl delete -f deployment.yaml
```

## ConfigMap과 Secret으로 설정 분리하기

하드코딩된 설정값을 컨테이너 이미지에 담으면 환경마다 다른 이미지를 빌드해야 합니다. ConfigMap과 Secret으로 설정을 외부에서 주입하면 하나의 이미지를 모든 환경에 쓸 수 있습니다.

```yaml
# ConfigMap: 일반 설정값
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DATABASE_HOST: "postgres-service"
  LOG_LEVEL: "info"

---
# Secret: 민감 정보 (base64 인코딩)
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
data:
  DATABASE_PASSWORD: cGFzc3dvcmQxMjM=
```

Deployment에서 환경 변수로 주입합니다.

```yaml
spec:
  containers:
  - name: app
    envFrom:
    - configMapRef:
        name: app-config
    - secretRef:
        name: app-secret
```

## HPA로 자동 확장 설정하기

Horizontal Pod Autoscaler(HPA)는 CPU나 메모리 사용량에 따라 Pod 수를 자동으로 조절합니다.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

CPU 사용률이 70%를 넘으면 Pod를 추가하고, 낮아지면 다시 줄입니다. 최소 2개, 최대 10개 범위 안에서 자동으로 조정됩니다.

## 로컬에서 시작하는 법

실제 클러스터 없이도 로컬에서 Kubernetes를 배울 수 있습니다.

| 도구 | 특징 |
|------|------|
| **minikube** | 로컬 단일 노드 클러스터. 입문자에게 적합 |
| **kind** | Docker 컨테이너로 K8s 클러스터 구성. CI 환경에서 유용 |
| **k3s** | 경량화 K8s. Raspberry Pi 등 저사양 환경에서도 동작 |
| **Docker Desktop** | Mac/Windows에서 K8s 옵션 활성화 가능 |

```bash
# minikube 시작
minikube start

# 대시보드 열기
minikube dashboard

# 로컬 이미지를 minikube에 로드
minikube image load my-app:latest
```

## 마치며

Kubernetes는 처음에는 개념이 많아 어렵게 느껴지지만, 핵심은 단순합니다. "원하는 상태(desired state)를 선언하면 K8s가 현재 상태를 그것에 맞추도록 계속 노력한다"는 철학입니다. Pod, Deployment, Service 세 가지 개념만 이해하면 실무에서 필요한 대부분의 작업을 처리할 수 있습니다.

Docker로 컨테이너화를 익혔다면, 다음 단계로 minikube로 로컬 클러스터를 띄워보고 간단한 앱을 배포해 보세요. 처음 `kubectl get pods`에서 Running 상태를 확인하는 순간, 그 강력함을 몸소 체감하게 될 것입니다.
