# Hybrid Nodes 네트워크 구성

> "온프레미스 노드가 AWS에 어떻게 연결되나요?"

EKS Hybrid Nodes의 네트워크 구성을 살펴봅니다.

## 전체 아키텍처

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│   AWS Cloud                                                         │
│   ┌─────────────────────────────────────────────────────────────┐  │
│   │  VPC: 10.0.0.0/16                                            │  │
│   │                                                              │  │
│   │  ┌─────────────────────────────────────────────────────┐    │  │
│   │  │  EKS Control Plane                                   │    │  │
│   │  │  Endpoint: xxx.eks.amazonaws.com                     │    │  │
│   │  └─────────────────────────────────────────────────────┘    │  │
│   │                          │                                   │  │
│   │  ┌───────────────────────┴───────────────────────────────┐  │  │
│   │  │  Private Subnets                                       │  │  │
│   │  │  10.0.1.0/24, 10.0.2.0/24                             │  │  │
│   │  │                                                        │  │  │
│   │  │  [EC2 Workers] (선택)                                  │  │  │
│   │  │                                                        │  │  │
│   │  │  ┌──────────────────────────────────────────────────┐ │  │  │
│   │  │  │  VPN Gateway 또는 Direct Connect                  │ │  │  │
│   │  │  └──────────────────────────────────────────────────┘ │  │  │
│   │  └────────────────────────┬──────────────────────────────┘  │  │
│   │                           │                                  │  │
│   └───────────────────────────┼──────────────────────────────────┘  │
│                               │                                     │
└───────────────────────────────┼─────────────────────────────────────┘
                                │
                    VPN Tunnel / Direct Connect
                                │
┌───────────────────────────────┼─────────────────────────────────────┐
│   On-Premises                 │                                     │
│                               │                                     │
│   ┌───────────────────────────┴─────────────────────────────────┐  │
│   │  Network: 192.168.0.0/16                                     │  │
│   │  Pod CIDR: 100.64.0.0/16 (VPC와 겹치지 않음)                │  │
│   │                                                              │  │
│   │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │  │
│   │  │ Hybrid Node1 │  │ Hybrid Node2 │  │ Hybrid Node3 │       │  │
│   │  │ 192.168.1.10 │  │ 192.168.1.11 │  │ 192.168.1.12 │       │  │
│   │  │              │  │              │  │              │       │  │
│   │  │ Pods:        │  │ Pods:        │  │ Pods:        │       │  │
│   │  │ 100.64.1.x   │  │ 100.64.2.x   │  │ 100.64.3.x   │       │  │
│   │  └──────────────┘  └──────────────┘  └──────────────┘       │  │
│   │                                                              │  │
│   └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## CIDR 계획

```
┌─────────────────────────────────────────────────────────────────┐
│                     CIDR 계획 예시                              │
│                                                                 │
│   AWS VPC:           10.0.0.0/16      (65,536 IPs)             │
│   ├── Subnet AZ-a:   10.0.1.0/24      (256 IPs)                │
│   ├── Subnet AZ-b:   10.0.2.0/24      (256 IPs)                │
│   └── Subnet AZ-c:   10.0.3.0/24      (256 IPs)                │
│                                                                 │
│   On-Prem Network:   192.168.0.0/16   (65,536 IPs)             │
│   ├── Servers:       192.168.1.0/24   (256 IPs)                │
│   └── Management:    192.168.100.0/24 (256 IPs)                │
│                                                                 │
│   Kubernetes:                                                   │
│   ├── Service CIDR:  172.20.0.0/16    (65,536 IPs)             │
│   └── Pod CIDR:      100.64.0.0/16    (65,536 IPs)             │
│       └── 겹치지 않도록 주의!                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## CNI 옵션

### VPC CNI (Remote Mode)

```yaml
# EKS Add-on으로 설치
apiVersion: eks.amazonaws.com/v1
kind: Addon
metadata:
  name: vpc-cni
spec:
  addonName: vpc-cni
  configuration: |
    enableNetworkPolicy: true
```

### Cilium

```yaml
# Cilium 설치 (Helm)
# values.yaml
ipam:
  mode: cluster-pool
  operator:
    clusterPoolIPv4PodCIDRList:
      - 100.64.0.0/16
    clusterPoolIPv4MaskSize: 24

encryption:
  enabled: true
  type: wireguard

hubble:
  enabled: true
  relay:
    enabled: true
```

## 로드밸런싱

### MetalLB for On-Premises

```yaml
# MetalLB 설치
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: on-prem-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.100.100-192.168.100.200
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: on-prem-l2
  namespace: metallb-system
spec:
  ipAddressPools:
  - on-prem-pool
```

