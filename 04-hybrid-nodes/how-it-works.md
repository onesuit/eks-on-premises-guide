# EKS Hybrid Nodes 작동 원리

> "온프레미스 서버가 어떻게 AWS EKS에 연결되는 거예요?"

EKS Hybrid Nodes의 작동 원리를 단계별로 살펴봅시다.

## 전체 아키텍처

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           AWS Cloud                                      │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │                         EKS Control Plane                          │  │
│  │   ┌─────────────────────────────────────────────────────────────┐ │  │
│  │   │                      API Server                              │ │  │
│  │   │   • 모든 API 요청 처리                                      │ │  │
│  │   │   • 인증/인가                                               │ │  │
│  │   │   • Admission Control                                       │ │  │
│  │   └──────────────────────────┬──────────────────────────────────┘ │  │
│  │                              │                                     │  │
│  │   ┌──────────────────────────┼──────────────────────────────────┐ │  │
│  │   │                          │                                   │ │  │
│  │   │  ┌───────────┐  ┌───────┴───────┐  ┌───────────────────┐   │ │  │
│  │   │  │ Scheduler │  │     etcd      │  │Controller Manager │   │ │  │
│  │   │  └───────────┘  └───────────────┘  └───────────────────┘   │ │  │
│  │   │                                                              │ │  │
│  │   └──────────────────────────────────────────────────────────────┘ │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                                    │                                     │
│  ┌─────────────────────────────────┼─────────────────────────────────┐  │
│  │               VPC               │                                 │  │
│  │   ┌─────────────────────────────┼─────────────────────────────┐  │  │
│  │   │      EKS Cluster Endpoint   │                             │  │  │
│  │   │      (Private or Public)    │                             │  │  │
│  │   └─────────────────────────────┼─────────────────────────────┘  │  │
│  │                                 │                                 │  │
│  │   ┌─────────────────────────────┼─────────────────────────────┐  │  │
│  │   │      VPN Gateway / DX       │                             │  │  │
│  │   └─────────────────────────────┼─────────────────────────────┘  │  │
│  └─────────────────────────────────┼─────────────────────────────────┘  │
└────────────────────────────────────┼────────────────────────────────────┘
                                     │
                          VPN Tunnel / Direct Connect
                                     │
┌────────────────────────────────────┼────────────────────────────────────┐
│                        On-Premises │                                    │
│  ┌─────────────────────────────────┼─────────────────────────────────┐ │
│  │                 Hybrid Node     │                                 │ │
│  │                                 │                                 │ │
│  │   ┌──────────────────────────┐  │  ┌──────────────────────────┐  │ │
│  │   │         kubelet          │◄─┘  │      kube-proxy          │  │ │
│  │   │                          │     │                          │  │ │
│  │   │  • API Server와 통신     │     │  • Service 네트워킹      │  │ │
│  │   │  • Pod 라이프사이클 관리 │     │  • iptables/IPVS 관리   │  │ │
│  │   └──────────────┬───────────┘     └──────────────────────────┘  │ │
│  │                  │                                                │ │
│  │   ┌──────────────┴───────────────────────────────────────────┐   │ │
│  │   │                    containerd                             │   │ │
│  │   │   ┌─────────┐  ┌─────────┐  ┌─────────┐                  │   │ │
│  │   │   │  Pod A  │  │  Pod B  │  │  Pod C  │                  │   │ │
│  │   │   └─────────┘  └─────────┘  └─────────┘                  │   │ │
│  │   └──────────────────────────────────────────────────────────┘   │ │
│  │                                                                   │ │
│  │   ┌──────────────────────────────────────────────────────────┐   │ │
│  │   │              SSM Agent 또는 IAM RA 인증서                │   │ │
│  │   │              (노드 인증용)                                │   │ │
│  │   └──────────────────────────────────────────────────────────┘   │ │
│  │                                                                   │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## 노드 등록 방식

Hybrid Node를 EKS에 등록하는 두 가지 방법이 있습니다.

### 방식 1: AWS Systems Manager (SSM)

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│    Hybrid Node                              AWS                     │
│   ┌────────────────┐                    ┌────────────────┐         │
│   │                │    Activation      │                │         │
│   │   SSM Agent    │◄──────────────────►│  SSM Service   │         │
│   │                │                    │                │         │
│   └───────┬────────┘                    └───────┬────────┘         │
│           │                                     │                   │
│           │         IAM Role 획득               │                   │
│           │◄────────────────────────────────────┤                   │
│           │                                     │                   │
│           │         EKS API 접근                │                   │
│           │─────────────────────────────────────►                   │
│           │                                                         │
└─────────────────────────────────────────────────────────────────────┘
```

**SSM 방식의 흐름:**

1. **SSM Hybrid Activation 생성**
   ```bash
   aws ssm create-activation \
     --iam-role HybridNodeRole \
     --registration-limit 10
   ```

2. **온프레미스 서버에 SSM Agent 설치 및 활성화**
   ```bash
   sudo amazon-ssm-agent -register \
     -code "activation-code" \
     -id "activation-id" \
     -region "ap-northeast-2"
   ```

3. **nodeadm으로 노드 등록**
   ```bash
   sudo nodeadm init -c file://nodeConfig.yaml
   ```

### 방식 2: IAM Roles Anywhere (IAM RA)

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│    Hybrid Node                              AWS                     │
│   ┌────────────────┐                    ┌────────────────┐         │
│   │                │     X.509 인증서    │  IAM Roles     │         │
│   │   X.509 Cert   │◄──────────────────►│   Anywhere     │         │
│   │   (PKI 발급)   │                    │                │         │
│   └───────┬────────┘                    └───────┬────────┘         │
│           │                                     │                   │
│           │      임시 IAM 자격증명 획득         │                   │
│           │◄────────────────────────────────────┤                   │
│           │                                     │                   │
│           │         EKS API 접근                │                   │
│           │─────────────────────────────────────►                   │
│           │                                                         │
└─────────────────────────────────────────────────────────────────────┘
```

**IAM RA 방식의 흐름:**

1. **Trust Anchor 생성** (CA 인증서 등록)
2. **IAM Role 생성** (노드용)
3. **Profile 생성** (Role과 Trust Anchor 연결)
4. **온프레미스 서버에 X.509 인증서 배포**
5. **nodeadm으로 노드 등록**

### 두 방식 비교

| 관점 | SSM | IAM Roles Anywhere |
|------|-----|-------------------|
| **설정 복잡도** | 낮음 | 높음 (PKI 필요) |
| **기존 PKI 활용** | ❌ | ✅ |
| **에이전트 필요** | SSM Agent | 인증서만 |
| **네트워크 요구** | SSM 엔드포인트 접근 | IAM RA 엔드포인트 접근 |
| **대규모 환경** | Activation 관리 필요 | PKI로 자동화 용이 |

## 네트워크 연결

### 필수 네트워크 구성

```
On-Premises                                    AWS
┌─────────────┐                          ┌─────────────┐
│             │                          │             │
│  Hybrid     │     VPN / Direct Connect │    VPC      │
│  Nodes      │◄────────────────────────►│             │
│             │                          │             │
└─────────────┘                          └─────────────┘

필요한 연결:
├── EKS API Server Endpoint (TCP 443)
├── SSM Endpoints (SSM 방식 사용 시)
├── ECR Endpoints (컨테이너 이미지)
├── CloudWatch Endpoints (로깅/메트릭)
└── S3 Endpoints (일부 기능에 필요)
```

### 네트워크 요구사항

```
온프레미스 → AWS 방향 (필수):

1. EKS API Server
   - eks.{region}.amazonaws.com:443
   - {cluster-id}.{region}.eks.amazonaws.com:443

2. SSM (SSM 방식 사용 시)
   - ssm.{region}.amazonaws.com:443
   - ssmmessages.{region}.amazonaws.com:443

3. ECR (컨테이너 이미지)
   - ecr.{region}.amazonaws.com:443
   - {account}.dkr.ecr.{region}.amazonaws.com:443

4. CloudWatch (선택)
   - logs.{region}.amazonaws.com:443
   - monitoring.{region}.amazonaws.com:443
```

### CNI 옵션

| CNI | 설명 | 장점 | 단점 |
|-----|------|------|------|
| **VPC CNI (Remote)** | AWS VPC와 통합 | AWS 네이티브, Pod에 VPC IP 할당 | Pod CIDR이 VPC CIDR과 겹치면 안 됨 |
| **Cilium** | eBPF 기반 CNI | 유연한 네트워크 정책, 높은 성능 | 학습 곡선 |

```yaml
# Cilium 사용 시 nodeConfig.yaml 예시
apiVersion: node.eks.aws/v1alpha1
kind: NodeConfig
spec:
  cluster:
    name: my-cluster
    region: ap-northeast-2
  hybrid:
    enableCredentialsFile: true
    iamRolesAnywhere:
      trustAnchorArn: arn:aws:rolesanywhere:...
      profileArn: arn:aws:rolesanywhere:...
      roleArn: arn:aws:iam::123456789:role/HybridNodeRole
  kubelet:
    config:
      maxPods: 110
```

## 통신 흐름

### kubectl → Pod 통신

```
사용자
  │
  │ kubectl exec -it pod-on-hybrid -- /bin/bash
  │
  ▼
┌─────────────────┐
│  EKS API Server │  (AWS Cloud)
│                 │
│  1. 인증/인가   │
│  2. Pod 위치 확인│
│  3. kubelet에 전달│
└────────┬────────┘
         │
    VPN/DX 터널
         │
         ▼
┌─────────────────┐
│  Hybrid Node    │  (On-Premises)
│                 │
│  kubelet        │
│    │            │
│    ▼            │
│  Pod Container  │
└─────────────────┘
```

### Pod → AWS 서비스 통신

```
┌──────────────────────────────────────────────────────────────────┐
│  Hybrid Node (On-Premises)                                       │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  Pod with Service Account                                │    │
│  │                                                          │    │
│  │  1. AWS SDK가 Pod Identity 웹훅 호출                    │    │
│  │  2. OIDC 토큰 획득                                      │    │
│  │  3. STS AssumeRoleWithWebIdentity                       │    │
│  │  4. 임시 자격증명으로 AWS 서비스 호출                   │    │
│  │                                                          │    │
│  └──────────────────────────┬───────────────────────────────┘    │
│                             │                                    │
└─────────────────────────────┼────────────────────────────────────┘
                              │
                         VPN/DX
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  AWS Services (S3, DynamoDB, SQS, etc.)                         │
└─────────────────────────────────────────────────────────────────┘
```

## 노드 설정 예시

### nodeConfig.yaml (SSM 방식)

```yaml
apiVersion: node.eks.aws/v1alpha1
kind: NodeConfig
spec:
  cluster:
    name: my-eks-cluster
    region: ap-northeast-2
    apiServerEndpoint: https://XXXXX.gr7.ap-northeast-2.eks.amazonaws.com
    certificateAuthority: LS0tLS1CRUdJTi...
    cidr: 10.100.0.0/16
  hybrid:
    ssm:
      activationId: "xxxxx-xxxx-xxxx"
      activationCode: "xxxxx"
  kubelet:
    config:
      maxPods: 110
      clusterDNS:
        - 172.20.0.10
    flags:
      - "--node-labels=eks.amazonaws.com/compute-type=hybrid"
      - "--register-with-taints=eks.amazonaws.com/compute-type=hybrid:NoSchedule"
```

### 노드 부트스트랩

```bash
# 1. nodeadm 설치
curl -OL https://hybrid-assets.eks.amazonaws.com/releases/latest/bin/linux/amd64/nodeadm
chmod +x nodeadm
sudo mv nodeadm /usr/local/bin/

# 2. 노드 초기화
sudo nodeadm init -c file://nodeConfig.yaml

# 3. 노드 상태 확인
sudo nodeadm status

# 4. kubelet 로그 확인
sudo journalctl -u kubelet -f
```

