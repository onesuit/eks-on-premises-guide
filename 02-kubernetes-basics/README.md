# Kubernetes 기초

> "Kubernetes가 뭔지는 알겠는데, 어떻게 돌아가는 건가요?"

이 장에서는 Kubernetes의 핵심 개념을 알아봅니다. 깊이 들어가기 전에, 큰 그림부터 그려봅시다.

## Kubernetes 전체 구조

Kubernetes 클러스터는 크게 두 부분으로 나뉩니다:

```
┌────────────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                          │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│   ┌──────────────────────────────────────────┐                 │
│   │            Control Plane                  │                 │
│   │   "클러스터의 두뇌"                       │                 │
│   │                                          │                 │
│   │   • 결정을 내림 (어디에 Pod를 배치할까?) │                 │
│   │   • 상태를 저장 (현재 무엇이 실행 중?)   │                 │
│   │   • API 제공 (kubectl 명령 처리)         │                 │
│   └──────────────────────────────────────────┘                 │
│                          │                                     │
│                          ▼                                     │
│   ┌──────────────────────────────────────────┐                 │
│   │           Worker Nodes                    │                 │
│   │   "실제 일을 하는 일꾼들"                 │                 │
│   │                                          │                 │
│   │   ┌────────┐ ┌────────┐ ┌────────┐      │                 │
│   │   │ Node 1 │ │ Node 2 │ │ Node 3 │      │                 │
│   │   │        │ │        │ │        │      │                 │
│   │   │ [Pod]  │ │ [Pod]  │ │ [Pod]  │      │                 │
│   │   │ [Pod]  │ │ [Pod]  │ │ [Pod]  │      │                 │
│   │   └────────┘ └────────┘ └────────┘      │                 │
│   └──────────────────────────────────────────┘                 │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### 비유로 이해하기

레스토랑으로 비유하면:

| Kubernetes | 레스토랑 |
|------------|---------|
| Control Plane | 주방장 + 매니저 |
| Worker Node | 요리사들 |
| Pod | 요리 (실제 음식) |
| Service | 메뉴 (손님이 주문하는 인터페이스) |
| Deployment | 레시피 (어떻게 요리할지 정의) |

주방장(Control Plane)은 직접 요리하지 않습니다. 대신:
- 주문을 받고 (API Server)
- 어떤 요리사가 여유 있는지 파악하고 (Scheduler)
- 요리사에게 지시하고 (Controller Manager)
- 레시피를 관리합니다 (etcd)

## 핵심 개념 미리보기

### Pod

Kubernetes에서 가장 작은 배포 단위입니다. 하나 이상의 컨테이너를 담고 있습니다.

```yaml
# 가장 단순한 Pod
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
  - name: app
    image: nginx:latest
```

> 💡 **참고**
>
> **왜 Container가 아니라 Pod인가요?**
>
> 함께 실행되어야 하는 컨테이너들이 있기 때문입니다. 예를 들어, 웹 서버와 그 로그를 수집하는 사이드카 컨테이너는 같은 Pod에서 실행됩니다.

### Service

Pod에 접근하기 위한 안정적인 주소입니다. Pod의 IP는 바뀔 수 있지만, Service의 IP는 고정됩니다.

### Deployment

Pod를 어떻게 배포하고 관리할지 정의합니다. 복제본 수, 업데이트 전략 등을 포함합니다.

### Namespace

클러스터 내에서 리소스를 논리적으로 분리합니다. 마치 폴더처럼요.

```
클러스터
├── namespace: default
├── namespace: production
│   ├── pod: api-server
│   └── pod: web-frontend
└── namespace: staging
    ├── pod: api-server
    └── pod: web-frontend
```

## 이 장에서 다룰 내용

1. **[핵심 개념 이해하기](core-concepts.md)**: Pod, Service, Deployment 등 기본 리소스
2. **[Control Plane 깊이 알아보기](control-plane.md)**: 클러스터의 두뇌가 어떻게 동작하는지
3. **[Worker Node 이해하기](worker-nodes.md)**: 실제 워크로드가 실행되는 곳

> ✅ **완료**
>
> **팁**: Kubernetes 경험이 있다면 이 장을 건너뛰고 [솔루션 비교](../03-comparison/README.md)로 바로 가도 됩니다.

