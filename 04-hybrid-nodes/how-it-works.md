# EKS Hybrid Nodes 작동 원리

> "온프레미스 서버가 어떻게 AWS EKS에 연결되는 거예요?"

## 먼저 알아두면 좋은 것: 실제로 얼마나 걸리나요?

처음 EKS Hybrid Nodes를 접하면 "복잡하겠다"는 생각이 듭니다. AWS 클라우드와 온프레미스를 연결한다니, 뭔가 대단한 작업이 필요할 것 같습니다.

실제로 해보면 생각보다 단순합니다. 이미 VPN이나 Direct Connect가 설정되어 있다면, 온프레미스 서버 한 대를 EKS 클러스터에 등록하는 데 **30분에서 1시간**이면 충분합니다.

과정은 이렇습니다:
1. AWS 콘솔에서 활성화 코드 발급 (5분)
2. 온프레미스 서버에 에이전트 설치 (10분)
3. 설정 파일 작성 후 노드 초기화 (15분)
4. `kubectl get nodes`에서 노드 확인 (완료!)

복잡해 보이는 것과 실제 복잡도는 다릅니다. 원리를 이해하면 더 쉬워집니다.

## 핵심 개념: 두뇌는 클라우드에, 손발은 온프레미스에

EKS Hybrid Nodes의 작동 원리를 이해하려면, 먼저 Kubernetes의 구조를 떠올려야 합니다.

Kubernetes 클러스터는 **Control Plane(두뇌)**과 **Worker Node(손발)**로 구성됩니다. Control Plane은 "어떤 Pod를 어디에 배치할지" 결정하고, Worker Node는 실제로 Pod를 실행합니다.

EKS Hybrid Nodes에서는:
- **Control Plane(API Server, etcd, Scheduler)**: AWS 클라우드에서 실행 → AWS가 관리
- **Worker Node(kubelet, containerd, Pods)**: 여러분의 온프레미스 서버에서 실행 → 여러분이 관리

이 둘은 VPN이나 Direct Connect로 연결되어, 마치 하나의 클러스터처럼 동작합니다.

## 작동 흐름: Pod가 생성되기까지

온프레미스 서버에 Pod가 배포되는 과정을 따라가 봅시다.

### 1단계: 사용자가 배포 요청

개발자가 `kubectl apply -f deployment.yaml` 명령을 실행합니다. 이 요청은 AWS 클라우드에 있는 **EKS API Server**로 전송됩니다.

```
사용자 (kubectl) ──────► AWS Cloud (EKS API Server)
```

### 2단계: API Server가 요청 처리

API Server는 세 가지 작업을 수행합니다:

1. **인증**: "이 사용자가 누구인지" (IAM 또는 OIDC로 확인)
2. **인가**: "이 사용자가 이 작업을 할 수 있는지" (RBAC으로 확인)
3. **저장**: 승인된 요청을 etcd에 기록

### 3단계: Scheduler가 노드 선택

Scheduler는 etcd에서 "아직 노드가 할당되지 않은 Pod"를 발견하고, **어떤 노드에 배치할지** 결정합니다.

이때 온프레미스의 Hybrid Node도 선택 대상에 포함됩니다. Scheduler는 리소스 여유, taint/toleration, node affinity 등을 고려해 최적의 노드를 선택합니다.

### 4단계: kubelet이 Pod 생성

선택된 노드의 **kubelet**이 API Server의 지시를 받아 실제로 Pod를 생성합니다. kubelet은 containerd에 명령을 내려 컨테이너를 시작합니다.

```
AWS Cloud                           On-Premises
┌─────────────────┐                ┌─────────────────┐
│  EKS API Server │                │  Hybrid Node    │
│                 │   VPN/DX       │                 │
│  "Pod를 Node1에 │ ─────────────► │  kubelet        │
│   배치하라"      │                │    ↓            │
│                 │                │  containerd     │
│                 │                │    ↓            │
│                 │                │  Pod 생성 완료! │
└─────────────────┘                └─────────────────┘
```

전체 과정에서 **API Server는 클라우드에 있지만, 실제 워크로드는 온프레미스에서 실행**됩니다.

## 노드 등록: 온프레미스 서버를 EKS에 연결하기

온프레미스 서버가 EKS 클러스터에 합류하려면 **"나는 신뢰할 수 있는 노드다"**를 증명해야 합니다. 두 가지 방법이 있습니다.

### 방식 1: SSM (Systems Manager) — 빠른 시작용

가장 간단한 방법입니다. AWS SSM Agent를 설치하고 활성화 코드만 입력하면 됩니다.

**어떻게 작동하나요?**

1. AWS 콘솔에서 "Hybrid Activation"을 생성하면 활성화 코드가 발급됩니다
2. 온프레미스 서버에 SSM Agent를 설치합니다
3. 활성화 코드를 입력하면 SSM Agent가 AWS와 통신하며 IAM 자격증명을 획득합니다
4. 이 자격증명으로 kubelet이 EKS API Server에 등록됩니다

**설정 예시:**

```bash
# 1. AWS에서 Hybrid Activation 생성
aws ssm create-activation \
  --iam-role HybridNodeRole \
  --registration-limit 10

# 결과: ActivationId와 ActivationCode 발급

# 2. 온프레미스 서버에서 SSM Agent 등록
sudo amazon-ssm-agent -register \
  -code "발급받은-activation-code" \
  -id "발급받은-activation-id" \
  -region "ap-northeast-2"

# 3. nodeadm으로 노드 초기화
sudo nodeadm init -c file://nodeConfig.yaml
```

**SSM 방식이 적합한 경우:**
- 빠르게 테스트하고 싶을 때
- PKI 인프라가 없을 때
- 소규모 환경(노드 수십 대 이하)

### 방식 2: IAM Roles Anywhere — 엔터프라이즈용

기존 PKI(공개키 인프라)를 보유한 기업에 적합합니다. X.509 인증서를 AWS에 등록하고, 인증서로 IAM 자격증명을 획득합니다.

**어떻게 작동하나요?**

1. 기업의 CA(인증기관) 인증서를 AWS에 "Trust Anchor"로 등록합니다
2. 각 서버에 CA가 발급한 X.509 인증서를 배포합니다
3. 서버는 인증서를 제시하고 AWS IAM Roles Anywhere로부터 임시 자격증명을 받습니다
4. 이 자격증명으로 EKS에 등록됩니다

**IAM RA 방식이 적합한 경우:**
- 기존 PKI 인프라를 활용하고 싶을 때
- 대규모 환경(노드 수백 대 이상)에서 자동화가 필요할 때
- 인증서 기반 관리가 회사 정책일 때

### 두 방식 비교

| 관점 | SSM | IAM Roles Anywhere |
|-----|-----|-------------------|
| **설정 복잡도** | 낮음 👍 | 높음 (PKI 필요) |
| **기존 PKI 활용** | ❌ | ✅ |
| **필요한 구성요소** | SSM Agent | 인증서만 |
| **대규모 자동화** | 활성화 코드 관리 필요 | PKI로 자동화 용이 👍 |
| **보안** | SSM 엔드포인트 의존 | 인증서 직접 통제 👍 |

> 💡 **추천**: 처음 시작한다면 **SSM**으로 빠르게 테스트하고, 프로덕션에서는 **IAM Roles Anywhere**를 검토하세요.

## 네트워크 연결: 클라우드와 온프레미스 사이

Hybrid Node가 정상적으로 작동하려면 **여러 AWS 서비스와 통신**해야 합니다. VPN이나 Direct Connect로 연결된 네트워크 경로가 필요합니다.

**흔히 겪는 문제**: 노드를 등록했는데 `NotReady` 상태가 지속된다면, 십중팔구 네트워크 문제입니다. 방화벽에서 특정 AWS 엔드포인트가 차단되어 있거나, DNS 해석이 안 되는 경우가 많습니다.

### 필수 연결 대상

온프레미스에서 다음 AWS 엔드포인트에 접근할 수 있어야 합니다:

| 엔드포인트 | 포트 | 용도 | 필수 여부 |
|-----------|-----|------|----------|
| EKS API Server | 443 | kubelet ↔ API Server 통신 | **필수** |
| SSM | 443 | 노드 등록 (SSM 방식) | 조건부 |
| IAM Roles Anywhere | 443 | 자격증명 획득 (RA 방식) | 조건부 |
| ECR | 443 | 컨테이너 이미지 풀 | **필수** |
| CloudWatch | 443 | 로그, 메트릭 전송 | 권장 |
| S3 | 443 | 일부 기능에 필요 | 권장 |

### 구체적인 엔드포인트 목록

```
# 필수 (모든 환경)
eks.{region}.amazonaws.com:443
{cluster-id}.{region}.eks.amazonaws.com:443
ecr.{region}.amazonaws.com:443
{account-id}.dkr.ecr.{region}.amazonaws.com:443

# SSM 방식 사용 시
ssm.{region}.amazonaws.com:443
ssmmessages.{region}.amazonaws.com:443

# 모니터링 (권장)
logs.{region}.amazonaws.com:443
monitoring.{region}.amazonaws.com:443
```

### CNI 선택: 네트워크 플러그인

Pod에 IP 주소를 할당하고 네트워크를 구성하는 CNI(Container Network Interface)를 선택해야 합니다.

**옵션 1: VPC CNI (Remote Mode)**

AWS 네이티브 CNI입니다. Pod에 VPC의 IP 주소가 직접 할당되어 AWS 서비스와 매끄럽게 통합됩니다.

- ✅ AWS 서비스와 원활한 통합
- ✅ Pod에서 VPC 리소스 직접 접근
- ⚠️ **주의**: Pod CIDR이 VPC CIDR과 겹치면 안 됨

**옵션 2: Cilium**

eBPF 기반의 고성능 CNI입니다. 네트워크 정책이 유연하고 관측성이 뛰어납니다.

- ✅ 강력한 Network Policy
- ✅ 높은 성능
- ✅ CIDR 제약 없음
- ⚠️ 학습 곡선 존재

> 💡 **추천**: AWS 서비스 통합이 중요하면 **VPC CNI**, 유연한 네트워크 정책이 필요하면 **Cilium**을 선택하세요.

## 통신 흐름 이해하기

### kubectl에서 Pod까지

개발자가 `kubectl exec`으로 Pod에 접속하면 어떤 경로로 연결될까요?

1. **kubectl → EKS API Server** (인터넷 또는 VPN 경유)
2. **API Server에서 인증/인가** (IAM 또는 OIDC)
3. **API Server → kubelet** (VPN/DX 경유, 온프레미스로)
4. **kubelet → Pod Container** (로컬)

```
개발자 PC
    │
    │ kubectl exec -it my-pod -- /bin/bash
    ▼
┌─────────────────────────────┐
│  EKS API Server (AWS)       │
│  1. 인증: "이 사용자 누구?" │
│  2. 인가: "exec 권한 있나?" │
│  3. 라우팅: "Pod가 어디에?" │
└────────────┬────────────────┘
             │
        VPN/Direct Connect
             │
             ▼
┌─────────────────────────────┐
│  Hybrid Node (On-Premises)  │
│  kubelet → containerd → Pod │
└─────────────────────────────┘
```

### Pod에서 AWS 서비스까지

온프레미스 Pod에서 S3에 파일을 업로드하면 어떻게 될까요?

1. **Pod 내 AWS SDK가 자격증명 요청**
2. **Pod Identity 또는 IRSA로 OIDC 토큰 획득**
3. **STS AssumeRoleWithWebIdentity로 임시 자격증명 획득**
4. **임시 자격증명으로 S3 API 호출**

모든 통신은 VPN/Direct Connect를 경유하여 AWS로 전달됩니다.

## 실제 설정 예시

### nodeConfig.yaml 작성

SSM 방식을 사용하는 경우의 설정 파일입니다:

```yaml
apiVersion: node.eks.aws/v1alpha1
kind: NodeConfig
spec:
  cluster:
    name: my-eks-cluster                    # EKS 클러스터 이름
    region: ap-northeast-2                  # 리전
    apiServerEndpoint: https://XXXXX.gr7.ap-northeast-2.eks.amazonaws.com
    certificateAuthority: LS0tLS1CRUdJTi...  # Base64 인코딩된 CA 인증서
    cidr: 10.100.0.0/16                     # Service CIDR
  
  hybrid:
    ssm:
      activationId: "xxxxx-xxxx-xxxx"       # SSM 활성화 ID
      activationCode: "xxxxx"               # SSM 활성화 코드
  
  kubelet:
    config:
      maxPods: 110                          # 노드당 최대 Pod 수
      clusterDNS:
        - 172.20.0.10                       # CoreDNS 주소
    flags:
      # Hybrid Node임을 표시하는 레이블
      - "--node-labels=eks.amazonaws.com/compute-type=hybrid"
      # 일반 워크로드가 스케줄되지 않도록 taint 설정 (선택)
      - "--register-with-taints=eks.amazonaws.com/compute-type=hybrid:NoSchedule"
```

### 노드 부트스트랩 절차

```bash
# 1. nodeadm 설치
curl -OL https://hybrid-assets.eks.amazonaws.com/releases/latest/bin/linux/amd64/nodeadm
chmod +x nodeadm
sudo mv nodeadm /usr/local/bin/

# 2. 설정 파일 준비
# nodeConfig.yaml을 작성하고 저장

# 3. 노드 초기화 (kubelet, containerd 등 자동 설치)
sudo nodeadm init -c file://nodeConfig.yaml

# 4. 노드 상태 확인
sudo nodeadm status

# 5. 클러스터에서 노드 확인
kubectl get nodes
# NAME                    STATUS   ROLES    AGE   VERSION
# hybrid-node-1           Ready    <none>   1m    v1.29.0-eks-...
```

## 다음 단계

노드 등록이 완료되면:

1. **워크로드 배포 테스트**: 간단한 nginx Pod를 배포해보세요
2. **네트워크 연결 확인**: Pod에서 AWS 서비스 접근이 되는지 테스트
3. **모니터링 설정**: CloudWatch Agent로 메트릭/로그 수집 구성

---

**다음으로**: [장점과 활용 사례](benefits.md)에서 어떤 상황에서 EKS Hybrid Nodes가 빛을 발하는지 알아보세요.
