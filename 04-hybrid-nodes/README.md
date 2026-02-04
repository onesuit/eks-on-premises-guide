# EKS Hybrid Nodes

> "AWS가 Control Plane을 관리해주고, 우리 서버를 Worker Node로 연결한다고요?"

네, 맞습니다. EKS Hybrid Nodes는 2024년 12월에 출시된 AWS의 최신 하이브리드 솔루션입니다. 기존 EKS 클러스터를 온프레미스까지 확장할 수 있게 해줍니다.

## EKS Hybrid Nodes란?

한 문장으로 정리하면:

> **AWS가 관리하는 EKS Control Plane에 온프레미스 서버를 Worker Node로 연결하는 것**

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│   AWS Cloud                                                         │
│   ┌─────────────────────────────────────────────────────────────┐  │
│   │                    EKS Control Plane                        │  │
│   │                    (AWS 완전 관리)                          │  │
│   │                                                             │  │
│   │   ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐      │  │
│   │   │  etcd   │  │   API   │  │Scheduler│  │Controller│      │  │
│   │   │         │  │ Server  │  │         │  │ Manager │      │  │
│   │   └─────────┘  └─────────┘  └─────────┘  └─────────┘      │  │
│   │                                                             │  │
│   └──────────────────────────┬──────────────────────────────────┘  │
│                              │                                     │
└──────────────────────────────┼─────────────────────────────────────┘
                               │
                    VPN / AWS Direct Connect
                               │
┌──────────────────────────────┼─────────────────────────────────────┐
│   On-Premises                │                                     │
│                              │                                     │
│   ┌──────────────────────────┴──────────────────────────────────┐ │
│   │                    Hybrid Worker Nodes                       │ │
│   │                    (고객이 관리)                             │ │
│   │                                                              │ │
│   │   ┌────────────┐  ┌────────────┐  ┌────────────┐           │ │
│   │   │  Server 1  │  │  Server 2  │  │  Server 3  │           │ │
│   │   │            │  │            │  │            │           │ │
│   │   │  [App A]   │  │  [App B]   │  │  [App C]   │           │ │
│   │   │  [App D]   │  │  [App E]   │  │  [App F]   │           │ │
│   │   └────────────┘  └────────────┘  └────────────┘           │ │
│   │                                                              │ │
│   └──────────────────────────────────────────────────────────────┘ │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## 왜 Hybrid Nodes인가?

### 기존 방식의 한계

**방식 1: 클라우드 전용 EKS**

```
❌ 온프레미스 워크로드를 클라우드로 옮길 수 없는 경우
❌ 데이터 레이턴시가 중요한 경우
❌ 규정상 데이터가 온프레미스에 있어야 하는 경우
```

**방식 2: EKS + 별도 온프레미스 K8s**

```
❌ 두 개의 다른 클러스터 관리
❌ 서로 다른 도구, 다른 운영 방식
❌ 워크로드 이동의 어려움
```

### Hybrid Nodes의 해결책

```
✅ 하나의 EKS 클러스터
✅ 동일한 도구 (kubectl, eksctl)
✅ 동일한 인증 (IAM)
✅ 동일한 모니터링 (CloudWatch)
✅ 워크로드를 어디서든 실행
```

## 핵심 가치 제안

### 1. 통합된 관리

```
kubectl get nodes

NAME                           STATUS   ROLES    AGE   VERSION
ip-10-0-1-100.ec2.internal    Ready    <none>   5d    v1.31
ip-10-0-2-101.ec2.internal    Ready    <none>   5d    v1.31
hybrid-node-001               Ready    <none>   2d    v1.31    ← 온프레미스
hybrid-node-002               Ready    <none>   2d    v1.31    ← 온프레미스
```

클라우드 노드와 온프레미스 노드가 같은 클러스터에 있습니다.

### 2. AWS 서비스 통합

```yaml
# Pod Identity로 AWS 서비스 접근
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/my-app-role
---
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  serviceAccountName: my-app
  nodeSelector:
    eks.amazonaws.com/compute-type: hybrid    # 온프레미스 노드에서 실행
  containers:
  - name: app
    image: my-app
    # AWS SDK가 자동으로 IAM 역할 사용
```

온프레미스에서 실행되는 Pod도 IAM 역할로 AWS 서비스에 접근할 수 있습니다.

### 3. Control Plane 부담 제거

| 항목 | 직접 운영 | EKS Hybrid Nodes |
|------|----------|------------------|
| etcd 백업 | 직접 구성 | AWS 자동 |
| API Server HA | 직접 구성 | AWS 관리 |
| 업그레이드 | 복잡한 절차 | 버튼 클릭 |
| 인증서 갱신 | 수동 | 자동 |
| 보안 패치 | 직접 적용 | AWS |

## 지원 환경

### 운영체제

| OS | 지원 여부 | 비고 |
|----|----------|------|
| Amazon Linux 2023 | ✅ | 권장 |
| Ubuntu 20.04 LTS | ✅ | |
| Ubuntu 22.04 LTS | ✅ | |
| RHEL 8 | ✅ | |
| RHEL 9 | ✅ | |

### 아키텍처

| 아키텍처 | 지원 여부 |
|----------|----------|
| x86_64 (AMD64) | ✅ |
| ARM64 (Graviton 호환) | ✅ |

### 최소 요구사항

```
하드웨어:
├── CPU: 1 vCPU 이상
├── Memory: 1GB 이상
├── Disk: 20GB 이상 (containerd, 이미지용)
└── Network: AWS endpoint 접근 가능

소프트웨어:
├── Container Runtime: containerd 1.7+
├── AWS SSM Agent 또는 IAM Roles Anywhere
└── 지원 OS 중 하나
```

## 이 장에서 다룰 내용

1. **[작동 원리](how-it-works.md)**: 노드가 어떻게 등록되고 통신하는지
2. **[장점과 활용 사례](benefits.md)**: 어떤 상황에서 가장 효과적인지
3. **[주의사항과 제약](considerations.md)**: 도입 전 알아야 할 것들

