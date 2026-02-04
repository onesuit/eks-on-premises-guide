# EKS Anywhere 작동 원리

> "처음부터 끝까지 직접 운영한다는 게 정확히 어떤 의미인가요?"

EKS Anywhere의 아키텍처와 작동 방식을 단계별로 살펴봅시다.

## 전체 아키텍처

### vSphere 환경 예시

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        On-Premises Data Center                          │
│                                                                         │
│   ┌─────────────────────────────────────────────────────────────────┐  │
│   │                        vCenter Server                            │  │
│   │                                                                  │  │
│   │   ┌─────────────────────────────────────────────────────────┐   │  │
│   │   │                  ESXi Hosts                              │   │  │
│   │   │                                                          │   │  │
│   │   │  ┌──────────────────────────────────────────────────┐   │   │  │
│   │   │  │         Management Cluster VMs                    │   │   │  │
│   │   │  │   ┌─────────┐                                    │   │   │  │
│   │   │  │   │ EKS-A   │  Cluster API Provider              │   │   │  │
│   │   │  │   │ Mgmt    │  CAPV (vSphere)                    │   │   │  │
│   │   │  │   │ Node    │                                    │   │   │  │
│   │   │  │   └────┬────┘                                    │   │   │  │
│   │   │  │        │                                         │   │   │  │
│   │   │  └────────┼─────────────────────────────────────────┘   │   │  │
│   │   │           │                                              │   │  │
│   │   │           │ Creates & Manages                            │   │  │
│   │   │           ▼                                              │   │  │
│   │   │  ┌──────────────────────────────────────────────────┐   │   │  │
│   │   │  │         Workload Cluster VMs                      │   │   │  │
│   │   │  │                                                   │   │   │  │
│   │   │  │  Control Plane:                                   │   │   │  │
│   │   │  │  ┌─────────┐ ┌─────────┐ ┌─────────┐             │   │   │  │
│   │   │  │  │ CP VM 1 │ │ CP VM 2 │ │ CP VM 3 │             │   │   │  │
│   │   │  │  │  etcd   │ │  etcd   │ │  etcd   │             │   │   │  │
│   │   │  │  │  API    │ │  API    │ │  API    │             │   │   │  │
│   │   │  │  └─────────┘ └─────────┘ └─────────┘             │   │   │  │
│   │   │  │                                                   │   │   │  │
│   │   │  │  Workers:                                         │   │   │  │
│   │   │  │  ┌─────────┐ ┌─────────┐ ┌─────────┐             │   │   │  │
│   │   │  │  │Worker 1 │ │Worker 2 │ │Worker N │             │   │   │  │
│   │   │  │  │ [Pods]  │ │ [Pods]  │ │ [Pods]  │             │   │   │  │
│   │   │  │  └─────────┘ └─────────┘ └─────────┘             │   │   │  │
│   │   │  │                                                   │   │   │  │
│   │   │  └──────────────────────────────────────────────────┘   │   │  │
│   │   │                                                          │   │  │
│   │   └──────────────────────────────────────────────────────────┘   │  │
│   │                                                                  │  │
│   └──────────────────────────────────────────────────────────────────┘  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## Cluster API 기반 아키텍처

EKS Anywhere는 Kubernetes Cluster API(CAPI)를 기반으로 합니다.

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│   Cluster API Architecture                                          │
│                                                                     │
│   ┌─────────────────────────────────────────────────────────────┐  │
│   │                    Management Cluster                        │  │
│   │                                                              │  │
│   │   ┌───────────────────────────────────────────────────────┐ │  │
│   │   │              Cluster API Controllers                   │ │  │
│   │   │                                                        │ │  │
│   │   │   ┌─────────────┐  ┌─────────────┐  ┌──────────────┐  │ │  │
│   │   │   │   Core      │  │  Bootstrap  │  │   Control    │  │ │  │
│   │   │   │   CAPI      │  │   Provider  │  │   Plane      │  │ │  │
│   │   │   │             │  │   (kubeadm) │  │   Provider   │  │ │  │
│   │   │   └─────────────┘  └─────────────┘  └──────────────┘  │ │  │
│   │   │                                                        │ │  │
│   │   │   ┌─────────────────────────────────────────────────┐ │ │  │
│   │   │   │          Infrastructure Provider                 │ │ │  │
│   │   │   │          (CAPV for vSphere)                      │ │ │  │
│   │   │   └─────────────────────────────────────────────────┘ │ │  │
│   │   │                                                        │ │  │
│   │   └───────────────────────────────────────────────────────┘ │  │
│   │                                                              │  │
│   │   Watches Cluster/Machine CRs → Creates Infrastructure      │  │
│   │                                                              │  │
│   └──────────────────────────────────────────────────────────────┘  │
│                              │                                      │
│                              │ Creates                              │
│                              ▼                                      │
│   ┌──────────────────────────────────────────────────────────────┐ │
│   │                    Workload Cluster                           │ │
│   └──────────────────────────────────────────────────────────────┘ │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 핵심 컴포넌트

| 컴포넌트 | 역할 |
|----------|------|
| **CAPI Core** | 클러스터/머신 리소스 관리 |
| **Bootstrap Provider** | 노드 초기화 (kubeadm 사용) |
| **Control Plane Provider** | Control Plane 라이프사이클 |
| **Infrastructure Provider** | 실제 인프라 생성 (CAPV, Tinkerbell 등) |

## 클러스터 생성 과정

### 1. 클러스터 구성 파일 작성

```yaml
# cluster-config.yaml
apiVersion: anywhere.eks.amazonaws.com/v1alpha1
kind: Cluster
metadata:
  name: my-cluster
spec:
  clusterNetwork:
    cniConfig:
      cilium: {}
    pods:
      cidrBlocks:
        - 192.168.0.0/16
    services:
      cidrBlocks:
        - 10.96.0.0/12
  controlPlaneConfiguration:
    count: 3                           # HA를 위해 3개
    endpoint:
      host: "192.168.1.100"           # API Server endpoint
    machineGroupRef:
      kind: VSphereMachineConfig
      name: my-cluster-cp
  datacenterRef:
    kind: VSphereDatacenterConfig
    name: my-cluster-datacenter
  kubernetesVersion: "1.31"
  workerNodeGroupConfigurations:
    - count: 3
      machineGroupRef:
        kind: VSphereMachineConfig
        name: my-cluster-worker
---
apiVersion: anywhere.eks.amazonaws.com/v1alpha1
kind: VSphereDatacenterConfig
metadata:
  name: my-cluster-datacenter
spec:
  datacenter: "Datacenter"
  network: "/Datacenter/network/VM Network"
  server: "vcenter.example.com"
  thumbprint: "AB:CD:EF:..."
---
apiVersion: anywhere.eks.amazonaws.com/v1alpha1
kind: VSphereMachineConfig
metadata:
  name: my-cluster-cp
spec:
  datastore: "/Datacenter/datastore/vsanDatastore"
  diskGiB: 50
  folder: "/Datacenter/vm/EKS"
  memoryMiB: 8192
  numCPUs: 4
  osFamily: bottlerocket
  resourcePool: "/Datacenter/host/Cluster/Resources"
  template: "/Datacenter/vm/Templates/bottlerocket-v1.31"
  users:
    - name: ec2-user
      sshAuthorizedKeys:
        - "ssh-rsa AAAA..."
```

### 2. 클러스터 생성 실행

```bash
# eksctl anywhere 설치
curl -fsSL https://anywhere.eks.amazonaws.com/releases/latest/eksctl-anywhere-linux-amd64.tar.gz | tar xz
sudo mv eksctl-anywhere /usr/local/bin/

# 클러스터 생성
eksctl anywhere create cluster -f cluster-config.yaml
```

### 3. 생성 과정 상세

```
eksctl anywhere create cluster
        │
        ▼
┌───────────────────────────────────────────────────────────────────┐
│  1. Bootstrap Cluster 생성                                        │
│     └── 로컬에 임시 Kind 클러스터 생성                           │
│                                                                   │
│  2. CAPI Controllers 설치                                         │
│     └── Bootstrap 클러스터에 CAPI 설치                           │
│                                                                   │
│  3. 인프라 생성                                                   │
│     └── vSphere에 VM 생성 (Control Plane + Worker)               │
│                                                                   │
│  4. Kubernetes 구성                                               │
│     ├── Control Plane 초기화                                     │
│     ├── etcd 클러스터 구성                                       │
│     └── Worker 노드 조인                                         │
│                                                                   │
│  5. 컴포넌트 설치                                                 │
│     ├── Cilium CNI                                               │
│     ├── CoreDNS                                                  │
│     └── EKS-A 컴포넌트                                           │
│                                                                   │
│  6. Management Cluster로 전환 (또는 Self-managed)                │
│     └── CAPI를 워크로드 클러스터로 이동                          │
│                                                                   │
│  7. Bootstrap Cluster 삭제                                        │
│     └── 임시 Kind 클러스터 정리                                  │
└───────────────────────────────────────────────────────────────────┘
```

## GitOps 통합 (Flux)

EKS Anywhere는 Flux를 기본으로 포함하여 GitOps 방식의 운영을 지원합니다.

```yaml
# cluster-config.yaml에 GitOps 추가
apiVersion: anywhere.eks.amazonaws.com/v1alpha1
kind: Cluster
metadata:
  name: my-cluster
spec:
  gitOpsRef:
    kind: FluxConfig
    name: my-flux-config
  # ... 나머지 설정
---
apiVersion: anywhere.eks.amazonaws.com/v1alpha1
kind: FluxConfig
metadata:
  name: my-flux-config
spec:
  github:
    owner: my-org
    repository: eks-anywhere-configs
    fluxSystemNamespace: flux-system
    branch: main
    clusterConfigPath: clusters/my-cluster
    personal: false
```

### GitOps 워크플로우

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│   Developer                    Git Repository                       │
│   ┌────────────┐              ┌────────────────────────────────┐   │
│   │            │   git push   │  clusters/my-cluster/          │   │
│   │   Changes  │─────────────►│  ├── apps/                     │   │
│   │            │              │  │   └── deployment.yaml       │   │
│   └────────────┘              │  └── infrastructure/           │   │
│                               │      └── namespace.yaml        │   │
│                               └────────────────┬───────────────┘   │
│                                                │                    │
│                                                │ Flux watches       │
│                                                ▼                    │
│                               ┌────────────────────────────────┐   │
│                               │   EKS Anywhere Cluster          │   │
│                               │                                 │   │
│                               │   ┌─────────────────────────┐  │   │
│                               │   │   Flux Controllers       │  │   │
│                               │   │   ├── source-controller  │  │   │
│                               │   │   ├── kustomize-ctrl     │  │   │
│                               │   │   └── helm-controller    │  │   │
│                               │   └────────────┬────────────┘  │   │
│                               │                │               │   │
│                               │                ▼ Apply         │   │
│                               │   ┌─────────────────────────┐  │   │
│                               │   │   Kubernetes Resources   │  │   │
│                               │   └─────────────────────────┘  │   │
│                               │                                 │   │
│                               └─────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## 업그레이드 프로세스

### 클러스터 업그레이드

```bash
# 업그레이드 실행
eksctl anywhere upgrade cluster -f cluster-config.yaml

# 또는 GitOps를 통한 업그레이드
# cluster-config.yaml의 kubernetesVersion 변경 후 git push
```

### 업그레이드 흐름

```
┌─────────────────────────────────────────────────────────────────────┐
│                        업그레이드 프로세스                          │
│                                                                     │
│  1. Control Plane 업그레이드 (Rolling)                              │
│     ┌─────────┐ ┌─────────┐ ┌─────────┐                            │
│     │ CP 1    │ │ CP 2    │ │ CP 3    │                            │
│     │ v1.30   │ │ v1.30   │ │ v1.30   │                            │
│     └────┬────┘ └─────────┘ └─────────┘                            │
│          │                                                          │
│          ▼                                                          │
│     ┌─────────┐ ┌─────────┐ ┌─────────┐                            │
│     │ CP 1    │ │ CP 2    │ │ CP 3    │                            │
│     │ v1.31 ✓ │ │ v1.30   │ │ v1.30   │                            │
│     └─────────┘ └────┬────┘ └─────────┘                            │
│                      │                                              │
│                      ▼                                              │
│     ... (반복) ...                                                  │
│                                                                     │
│  2. Worker Node 업그레이드 (Rolling)                                │
│     └── 한 번에 하나씩 drain → upgrade → uncordon                  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## Bare Metal (Tinkerbell) 환경

베어메탈 환경에서는 Tinkerbell을 사용합니다.

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Bare Metal Environment                          │
│                                                                     │
│   ┌─────────────────────────────────────────────────────────────┐  │
│   │                   Tinkerbell Stack                           │  │
│   │                                                              │  │
│   │   ┌────────────┐  ┌────────────┐  ┌────────────┐           │  │
│   │   │   Boots    │  │   Hegel    │  │  Tink      │           │  │
│   │   │   (DHCP,   │  │ (Metadata) │  │  Server    │           │  │
│   │   │    PXE)    │  │            │  │            │           │  │
│   │   └────────────┘  └────────────┘  └────────────┘           │  │
│   │                                                              │  │
│   └─────────────────────────────────────────────────────────────┘  │
│                              │                                      │
│                              │ PXE Boot & Provision                 │
│                              ▼                                      │
│   ┌─────────────────────────────────────────────────────────────┐  │
│   │                   Physical Servers                           │  │
│   │                                                              │  │
│   │   ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │  │
│   │   │Server 1  │  │Server 2  │  │Server 3  │  │Server 4  │   │  │
│   │   │(CP)      │  │(CP)      │  │(Worker)  │  │(Worker)  │   │  │
│   │   └──────────┘  └──────────┘  └──────────┘  └──────────┘   │  │
│   │                                                              │  │
│   └─────────────────────────────────────────────────────────────┘  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Tinkerbell 워크플로우

```yaml
# hardware.yaml
apiVersion: tinkerbell.org/v1alpha1
kind: Hardware
metadata:
  name: server-01
spec:
  id: "abc123"
  metadata:
    facility:
      facility_code: dc1
    instance:
      hostname: server-01
      id: "abc123"
  interfaces:
    - dhcp:
        mac: "00:50:56:ab:cd:ef"
        hostname: server-01
```

---

[다음: 장점과 활용 사례 →](benefits.md)
