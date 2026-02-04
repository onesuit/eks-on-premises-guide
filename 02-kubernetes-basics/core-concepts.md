# 핵심 개념 이해하기

> "Pod, Service, Deployment... 비슷해 보이는데 뭐가 다른 건가요?"

Kubernetes를 처음 접하면 수많은 용어에 압도당합니다. 하나씩 차근차근 알아봅시다.

## Pod: 가장 작은 배포 단위

### Pod가 뭔가요?

Pod는 하나 이상의 컨테이너를 담는 "포장 상자"입니다.

```
┌─────────────────────────────────────┐
│              Pod                     │
│  ┌───────────────────────────────┐  │
│  │     Container (nginx)          │  │
│  └───────────────────────────────┘  │
│                                     │
│  - IP 주소: 10.0.0.15              │
│  - Volume: /data                   │
└─────────────────────────────────────┘
```

### 왜 Container가 아니라 Pod인가요?

때로는 여러 컨테이너가 함께 동작해야 합니다:

```
┌─────────────────────────────────────────────────┐
│                    Pod                           │
│  ┌─────────────────┐  ┌─────────────────┐       │
│  │  App Container  │  │  Sidecar        │       │
│  │                 │  │  (Log Collector)│       │
│  │  nginx:latest   │  │  fluentd        │       │
│  └────────┬────────┘  └────────┬────────┘       │
│           │                    │                │
│           └────────┬───────────┘                │
│                    │                            │
│           ┌────────▼────────┐                   │
│           │  Shared Volume  │                   │
│           │   /var/log      │                   │
│           └─────────────────┘                   │
└─────────────────────────────────────────────────┘
```

같은 Pod 안의 컨테이너들은:
- 같은 IP 주소 공유
- localhost로 서로 통신 가능
- 볼륨 공유 가능

### Pod YAML 예시

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
  labels:                      # 이 Pod를 식별하는 라벨
    app: my-app
    version: v1
spec:
  containers:
  - name: app
    image: my-app:1.0.0
    ports:
    - containerPort: 8080
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```

> ⚠️ **주의**
>
> **주의**: Pod를 직접 생성하는 것은 권장되지 않습니다. Pod가 죽으면 자동으로 재생성되지 않기 때문입니다. 대신 Deployment를 사용하세요.

## Service: 안정적인 접근점

### 문제 상황

Pod의 IP는 고정되지 않습니다:

```
시간 T1:
  Pod A (IP: 10.0.0.15) ← App B가 이 IP로 접속

시간 T2: Pod A 재시작
  Pod A (IP: 10.0.0.87) ← 새 IP! App B는 여전히 10.0.0.15로 접속 시도
                          → 연결 실패!
```

### Service의 해결책

Service는 Pod 앞에 고정된 주소를 제공합니다:

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  App B ──────► Service (my-app-svc) ──────► Pod A          │
│                IP: 10.96.0.100              IP: 변동       │
│                Port: 80                                     │
│                                                             │
│                └── Pod가 재시작되어도 Service IP는 고정!   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Service 타입

| 타입 | 용도 | 접근 범위 |
|------|------|----------|
| **ClusterIP** | 클러스터 내부 통신 | 클러스터 내부만 |
| **NodePort** | 외부 접근 (개발용) | 노드IP:포트로 접근 |
| **LoadBalancer** | 외부 접근 (운영용) | 클라우드 LB 사용 |

### Service YAML 예시

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-svc
spec:
  selector:
    app: my-app           # label이 app=my-app인 Pod를 찾음
  ports:
  - port: 80              # Service 포트
    targetPort: 8080      # Pod의 컨테이너 포트
  type: ClusterIP
```

> 💡 **참고**
>
> **Service Discovery**
>
> Pod는 Service 이름으로 다른 서비스에 접근할 수 있습니다:
> ```
> curl http://my-app-svc:80
> ```
> Kubernetes DNS가 자동으로 IP를 찾아줍니다.

## Deployment: 선언적 배포

### 문제 상황

"nginx Pod를 3개 유지하고 싶어요"

직접 관리한다면:
1. Pod 3개 생성
2. 하나가 죽으면? 수동으로 다시 생성
3. 업데이트하려면? 하나씩 삭제하고 다시 생성

### Deployment의 해결책

원하는 상태만 선언하면 Kubernetes가 알아서 유지합니다:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3                    # "항상 3개 유지해주세요"
  selector:
    matchLabels:
      app: my-app
  template:                      # Pod 템플릿
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: app
        image: my-app:1.0.0
        ports:
        - containerPort: 8080
```

### Deployment가 해주는 것들

```
┌─────────────────────────────────────────────────────────────┐
│                     Deployment                               │
│  "replicas: 3을 유지하라"                                   │
│                                                             │
│  현재 상태                    원하는 상태                   │
│  ┌─────┐ ┌─────┐             ┌─────┐ ┌─────┐ ┌─────┐       │
│  │Pod 1│ │Pod 2│      →      │Pod 1│ │Pod 2│ │Pod 3│       │
│  └─────┘ └─────┘             └─────┘ └─────┘ └─────┘       │
│                                                             │
│  "Pod 2개밖에 없네? Pod 3 생성!"                            │
└─────────────────────────────────────────────────────────────┘
```

1. **자동 복구**: Pod가 죽으면 자동 재생성
2. **롤링 업데이트**: 무중단 배포
3. **롤백**: 문제 시 이전 버전으로 복구
4. **스케일링**: replicas 수 조정으로 확장/축소

### 업데이트 과정

```
버전 v1 → v2 업데이트:

1단계: v1 v1 v1        (현재 상태)
2단계: v1 v1 v2        (v2 하나 추가)
3단계: v1 v2 v2        (v1 하나 제거, v2 하나 추가)
4단계: v2 v2 v2        (완료)
```

## ConfigMap과 Secret

### ConfigMap: 설정 분리

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DATABASE_HOST: "db.example.com"
  LOG_LEVEL: "info"
```

Pod에서 사용:

```yaml
spec:
  containers:
  - name: app
    envFrom:
    - configMapRef:
        name: app-config
```

### Secret: 민감 정보 관리

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
data:
  PASSWORD: cGFzc3dvcmQxMjM=    # base64 인코딩
```

> ⚠️ **주의**
>
> **주의**: Secret은 base64 인코딩일 뿐, 암호화되지 않습니다. 프로덕션에서는 외부 시크릿 관리 도구(Vault, AWS Secrets Manager 등)를 사용하세요.

## Namespace: 논리적 분리

하나의 클러스터를 여러 환경으로 나눕니다:

```
클러스터
├── namespace: default        # 기본
├── namespace: kube-system    # 시스템 컴포넌트
├── namespace: production     # 프로덕션 앱
├── namespace: staging        # 스테이징 앱
└── namespace: development    # 개발 앱
```

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
```

```bash
# production namespace에 배포
kubectl apply -f deployment.yaml -n production

# staging namespace에 배포
kubectl apply -f deployment.yaml -n staging
```

## 개념 간의 관계

```
┌─────────────────────────────────────────────────────────────────┐
│                      Namespace                                   │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                    Deployment                              │  │
│  │  ┌─────────────────────────────────────────────────────┐  │  │
│  │  │                ReplicaSet                            │  │  │
│  │  │  ┌─────────┐  ┌─────────┐  ┌─────────┐             │  │  │
│  │  │  │  Pod 1  │  │  Pod 2  │  │  Pod 3  │             │  │  │
│  │  │  └─────────┘  └─────────┘  └─────────┘             │  │  │
│  │  └─────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────┘  │
│                          ▲                                       │
│                          │ selector로 연결                       │
│                          │                                       │
│  ┌───────────────────────┴───────────────────────────────────┐  │
│  │                    Service                                 │  │
│  │  (고정 IP로 위 Pod들에 트래픽 분배)                       │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌─────────────────┐  ┌─────────────────┐                       │
│  │   ConfigMap     │  │     Secret      │                       │
│  │  (설정 값)       │  │  (민감 정보)     │                       │
│  └─────────────────┘  └─────────────────┘                       │
└─────────────────────────────────────────────────────────────────┘
```

---

[다음: Control Plane 깊이 알아보기 →](control-plane.md)
