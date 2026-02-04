# Worker Node 이해하기

> "Control Plane이 두뇌라면, Worker Node는 실제로 일하는 손과 발입니다."

Worker Node는 실제 애플리케이션 컨테이너가 실행되는 곳입니다. 우리가 배포하는 모든 Pod는 Worker Node에서 돌아갑니다.

## Worker Node 구성 요소

```
┌──────────────────────────────────────────────────────────────────┐
│                         Worker Node                               │
│                                                                   │
│   ┌───────────────────────────────────────────────────────────┐  │
│   │                        kubelet                             │  │
│   │   "Control Plane의 명령을 받아 컨테이너를 관리하는 에이전트" │  │
│   └───────────────────────────────────────────────────────────┘  │
│                              │                                    │
│                              ▼                                    │
│   ┌───────────────────────────────────────────────────────────┐  │
│   │                  Container Runtime                         │  │
│   │   "실제로 컨테이너를 실행하는 엔진 (containerd, CRI-O)"    │  │
│   └───────────────────────────────────────────────────────────┘  │
│                              │                                    │
│                              ▼                                    │
│   ┌────────────┐ ┌────────────┐ ┌────────────┐                   │
│   │  Pod (App) │ │  Pod (DB)  │ │  Pod (API) │                   │
│   └────────────┘ └────────────┘ └────────────┘                   │
│                                                                   │
│   ┌───────────────────────────────────────────────────────────┐  │
│   │                      kube-proxy                            │  │
│   │   "Service 네트워킹을 담당하는 네트워크 프록시"            │  │
│   └───────────────────────────────────────────────────────────┘  │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

## 각 구성 요소 설명

### 1. kubelet: 노드의 관리자

kubelet은 각 노드에서 실행되는 에이전트입니다. Control Plane과 통신하며 Pod의 라이프사이클을 관리합니다.

```
                Control Plane
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│                     kubelet                              │
│                                                         │
│   주요 역할:                                            │
│   1. API Server에 노드 등록                             │
│   2. Pod 스펙 받아서 컨테이너 생성/삭제                 │
│   3. 컨테이너 상태 모니터링                             │
│   4. 상태를 API Server에 보고                           │
│   5. 리소스 사용량 수집 (metrics)                       │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**kubelet의 동작 방식:**

```
1. API Server에서 이 노드에 할당된 Pod 목록 watch

2. 변경 감지 시:
   - 새 Pod 추가됨 → Container Runtime에 컨테이너 생성 요청
   - Pod 삭제됨 → Container Runtime에 컨테이너 삭제 요청
   - Pod 변경됨 → 필요한 조치 수행

3. 주기적으로:
   - 컨테이너 상태 확인 (liveness/readiness probe)
   - 상태를 API Server에 보고
```

> 💡 **참고**
>
> **Probe란?**
>
> kubelet이 컨테이너 상태를 확인하는 방법입니다:
>
> - **Liveness Probe**: 컨테이너가 살아있는지 확인. 실패 시 재시작.
> - **Readiness Probe**: 트래픽 받을 준비가 됐는지 확인. 실패 시 Service에서 제외.
> - **Startup Probe**: 시작 완료됐는지 확인. 완료 전까지 다른 probe 비활성화.

```yaml
# Probe 설정 예시
spec:
  containers:
  - name: app
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 3
      periodSeconds: 10
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
```

### 2. Container Runtime: 컨테이너 실행 엔진

Container Runtime은 실제로 컨테이너를 생성하고 실행하는 소프트웨어입니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   kubelet ──── CRI (Container Runtime Interface) ──── Runtime  │
│                                                                 │
│                              │                                  │
│                              ▼                                  │
│                    ┌─────────────────┐                          │
│                    │   containerd    │  ← 가장 많이 사용       │
│                    │   또는 CRI-O    │  ← Red Hat 계열에서 사용│
│                    └─────────────────┘                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**주요 Container Runtime:**

| Runtime | 특징 |
|---------|------|
| **containerd** | Docker에서 분리, CNCF 프로젝트, 가장 범용 |
| **CRI-O** | Kubernetes 전용으로 설계, 가벼움 |

> ⚠️ **주의**
>
> **Docker는 어디 갔나요?**
>
> Kubernetes 1.24부터 Docker (dockershim)는 더 이상 지원되지 않습니다. containerd나 CRI-O를 사용해야 합니다. Docker로 빌드한 이미지는 여전히 사용 가능합니다!

### 3. kube-proxy: 네트워크 마법사

kube-proxy는 Service의 네트워킹을 담당합니다.

```
외부 요청: "my-service:80으로 보내주세요"

┌─────────────────────────────────────────────────────────────────┐
│                        kube-proxy                                │
│                                                                 │
│   Service: my-service (10.96.0.100:80)                         │
│                    │                                            │
│                    ▼                                            │
│   ┌─────────────────────────────────────────┐                  │
│   │         iptables / IPVS 규칙            │                  │
│   │                                         │                  │
│   │  10.96.0.100:80 → 10.0.0.15:8080       │                  │
│   │                 → 10.0.0.16:8080       │  로드밸런싱      │
│   │                 → 10.0.0.17:8080       │                  │
│   └─────────────────────────────────────────┘                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**kube-proxy 모드:**

| 모드 | 설명 | 성능 |
|------|------|------|
| **iptables** | 기본 모드, 리눅스 iptables 사용 | 보통 |
| **IPVS** | 리눅스 커널 로드밸런서 사용 | 높음 |
| **userspace** | 레거시 모드, 거의 사용 안 함 | 낮음 |

## Node 리소스 관리

### 리소스 예약

노드의 모든 리소스를 Pod에 줄 수 없습니다. 시스템 운영을 위한 리소스가 필요합니다:

```
노드 전체 리소스: 16 CPU, 64GB Memory
┌────────────────────────────────────────────────────────────────┐
│                                                                │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │  Allocatable (Pod에 할당 가능)                           │ │
│  │  14.5 CPU, 56GB Memory                                   │ │
│  │  ┌────────────┐ ┌────────────┐ ┌────────────┐           │ │
│  │  │   Pod 1    │ │   Pod 2    │ │   Pod 3    │           │ │
│  │  └────────────┘ └────────────┘ └────────────┘           │ │
│  └──────────────────────────────────────────────────────────┘ │
│                                                                │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │  Reserved (시스템용)                                     │ │
│  │  - kube-reserved: kubelet, container runtime용           │ │
│  │  - system-reserved: OS 프로세스용                        │ │
│  │  - eviction-threshold: 여유 공간                         │ │
│  │  총 1.5 CPU, 8GB Memory                                  │ │
│  └──────────────────────────────────────────────────────────┘ │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### Requests vs Limits

Pod가 사용할 리소스를 정의하는 두 가지 방법:

```yaml
resources:
  requests:         # 최소 보장량 (스케줄링에 사용)
    memory: "256Mi"
    cpu: "500m"
  limits:           # 최대 사용량 (초과 시 제한/종료)
    memory: "512Mi"
    cpu: "1000m"
```

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  Memory 사용량                                                  │
│                                                                 │
│  512Mi ──────────────────── Limit (초과 시 OOM Kill)           │
│         █████████████████                                       │
│         █████████████████                                       │
│  256Mi ──────────────────── Request (이만큼은 보장)            │
│         █████████████████                                       │
│      0 ─────────────────────                                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> 💡 **참고**
>
> **Request와 Limit 설정 팁**
>
> - **Request**: 일반적인 사용량 기준
> - **Limit**: 피크 사용량 또는 Request의 2배 정도
> - Memory는 초과 시 OOM Kill, CPU는 throttling
> - Request 없이 Limit만 설정하면 Request = Limit

## Node 상태 확인

```bash
# 노드 목록 및 상태
kubectl get nodes

# 노드 상세 정보
kubectl describe node <node-name>
```

**Node Conditions:**

| Condition | 의미 |
|-----------|------|
| Ready | 노드가 정상이고 Pod 실행 가능 |
| MemoryPressure | 메모리 부족 |
| DiskPressure | 디스크 공간 부족 |
| PIDPressure | 프로세스 수 한계 |
| NetworkUnavailable | 네트워크 설정 미완료 |

```
# kubectl describe node 출력 예시
Conditions:
  Type             Status
  ----             ------
  Ready            True      ← 정상!
  MemoryPressure   False     ← 문제 없음
  DiskPressure     False     ← 문제 없음
  PIDPressure      False     ← 문제 없음
```

## 온프레미스 Worker Node의 특수성

클라우드 환경과 달리 온프레미스 Worker Node는 몇 가지 추가 고려사항이 있습니다:

### 1. 하드웨어 다양성

```
클라우드:
  모든 노드가 동일한 스펙 (예: m5.xlarge)

온프레미스:
  Node 1: Dell R640 (Intel Xeon, 128GB RAM)
  Node 2: HP ProLiant (AMD EPYC, 64GB RAM)
  Node 3: 오래된 서버 (32GB RAM)
```

라벨을 활용한 관리:

```yaml
# 노드에 라벨 추가
kubectl label node node-1 hardware-generation=new
kubectl label node node-3 hardware-generation=old

# Pod에서 특정 하드웨어 선택
spec:
  nodeSelector:
    hardware-generation: new
```

### 2. 네트워크 환경

온프레미스는 네트워크 설정이 더 복잡할 수 있습니다:

- VLAN 설정
- 방화벽 규칙
- DNS 설정
- Proxy 환경

### 3. 스토리지

클라우드는 EBS 같은 매니지드 스토리지가 있지만, 온프레미스는:

- NFS
- iSCSI
- Ceph
- 로컬 디스크

## EKS Hybrid Nodes vs EKS Anywhere의 Node

| 관점 | EKS Hybrid Nodes | EKS Anywhere |
|------|------------------|--------------|
| Node 위치 | 온프레미스 | 온프레미스 |
| Node 등록 | AWS Systems Manager로 자동 | EKS Anywhere CLI로 관리 |
| CNI | Amazon VPC CNI 또는 Cilium | Cilium |
| Node 모니터링 | CloudWatch Agent | 직접 구성 |

다음 장에서 두 솔루션을 자세히 비교해 보겠습니다.

---

[다음: EKS Hybrid Nodes vs EKS Anywhere 비교 →](../03-comparison/README.md)
