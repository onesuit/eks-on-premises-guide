# 자주 묻는 질문 (FAQ)

> 고객들이 가장 궁금해하는 질문들을 모았습니다.

## 목차
- [인증 및 접근 관리](#인증-및-접근-관리)
- [네트워크 및 보안](#네트워크-및-보안)
- [운영 및 관리 도구](#운영-및-관리-도구)
- [비용 및 라이선스](#비용-및-라이선스)
- [기타](#기타)

---

## 인증 및 접근 관리

### IAM Roles Anywhere와 SSM Hybrid Activations는 무엇이고, 어떻게 동작하나요?

온프레미스 서버가 AWS 리소스에 접근하려면 "이 서버가 누구인지" 증명해야 합니다. AWS 밖에 있는 서버가 IAM 역할을 어떻게 사용할 수 있을까요?

**문제 상황:**

```
EC2 인스턴스:
└── 인스턴스 메타데이터 서비스에서 자동으로 IAM 역할 획득
    (169.254.169.254에 요청하면 됨)

온프레미스 서버:
└── 인스턴스 메타데이터 서비스가 없음!
    IAM 역할을 어떻게 얻지?
```

**해결책 1: SSM Hybrid Activations**

AWS Systems Manager를 사용하여 온프레미스 서버를 "관리 대상"으로 등록합니다.

```
작동 방식:

1. AWS 콘솔에서 Hybrid Activation 생성
   └── Activation Code와 ID 발급

2. 온프레미스 서버에 SSM Agent 설치 및 등록
   $ sudo amazon-ssm-agent -register \
       -code "activation-code" \
       -id "activation-id" \
       -region "ap-northeast-2"

3. SSM Agent가 AWS와 통신하여 IAM 역할 획득
   └── 이제 이 서버는 IAM 역할로 AWS 서비스 접근 가능

4. EKS Hybrid Node로 등록
   └── nodeadm이 SSM을 통해 인증하여 EKS 클러스터에 조인
```

**해결책 2: IAM Roles Anywhere**

기존에 가지고 있는 PKI(공개키 인프라)의 인증서를 사용합니다.

```
작동 방식:

1. 기존 CA(Certificate Authority) 또는 새로 생성한 CA를 
   AWS IAM Roles Anywhere에 "Trust Anchor"로 등록

2. 온프레미스 서버에 해당 CA가 발급한 X.509 인증서 배포

3. 서버가 인증서를 사용하여 AWS STS에 임시 자격증명 요청
   └── "이 인증서를 가진 서버는 이 IAM 역할을 사용할 수 있어"

4. 임시 자격증명으로 AWS 서비스 접근

장점:
├── 기존 PKI 인프라 활용 가능
├── SSM Agent 불필요
└── 대규모 환경에서 인증서 관리가 더 용이할 수 있음
```

**비교:**

| 방식 | 장점 | 단점 |
|------|------|------|
| SSM Hybrid Activations | 설정 간단, PKI 불필요 | 서버마다 등록 필요 |
| IAM Roles Anywhere | 기존 PKI 활용, 자동화 용이 | 초기 설정 복잡 |

---

### IRSA(IAM Roles for Service Accounts)는 왜 쓰나요?

**IRSA 없이 운영할 때의 문제:**

```
문제 상황: Pod에서 S3에 접근해야 함

옵션 1: Access Key를 Secret으로 저장
├── 보안 위험: Key가 유출되면 큰 문제
├── Key 로테이션이 어려움
└── 모든 Pod가 같은 Key 공유

apiVersion: v1
kind: Secret
metadata:
  name: aws-credentials
stringData:
  AWS_ACCESS_KEY_ID: "AKIA..."     # 유출 위험!
  AWS_SECRET_ACCESS_KEY: "..."     # 영구 자격증명!

옵션 2: EC2 인스턴스 역할 사용
├── 그 노드의 모든 Pod가 같은 권한 획득
├── 최소 권한 원칙 위반
└── Pod A는 S3만 필요한데 DynamoDB 권한도 가짐
```

**IRSA로 해결:**

```
IRSA 적용 후:

1. ServiceAccount에 IAM 역할 연결
apiVersion: v1
kind: ServiceAccount
metadata:
  name: s3-reader
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/S3ReadOnly

2. Pod에서 해당 ServiceAccount 사용
apiVersion: v1
kind: Pod
spec:
  serviceAccountName: s3-reader  # 이 Pod만 S3 읽기 권한
  containers:
  - name: app
    image: my-app
    # AWS SDK가 자동으로 임시 자격증명 획득

장점:
├── Pod별로 다른 IAM 역할 가능 (최소 권한 원칙)
├── 임시 자격증명 사용 (자동 로테이션)
├── Access Key 관리 불필요
└── OIDC 기반으로 안전
```

**작동 원리:**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  Pod 시작 시:                                                   │
│  1. Kubernetes가 ServiceAccount 토큰(JWT) 생성                 │
│  2. Pod에 토큰 마운트 (/var/run/secrets/eks.amazonaws.com/)    │
│                                                                 │
│  AWS SDK가 AWS 서비스 호출 시:                                 │
│  3. JWT 토큰을 AWS STS에 전달                                  │
│  4. STS가 EKS OIDC Provider에 토큰 검증 요청                   │
│  5. 검증 성공 시 임시 자격증명 발급                            │
│  6. 임시 자격증명으로 S3/DynamoDB 등 접근                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

### RBAC은 어떻게 동작하나요?

RBAC(Role-Based Access Control)은 "누가 무엇을 할 수 있는지" 정의합니다.

**핵심 개념:**

```
RBAC 구성요소:

Subject (누가)          + Verb (무엇을)     + Resource (대상)
├── User               ├── get             ├── pods
├── Group              ├── list            ├── deployments
└── ServiceAccount     ├── create          ├── secrets
                       ├── delete          └── nodes
                       └── watch
```

**실제 예시:**

```yaml
# 1. Role 정의: "production 네임스페이스에서 Pod 읽기 권한"
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: production
  name: pod-reader
rules:
- apiGroups: [""]           # "" = core API (pods, services 등)
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get"]            # Pod 로그도 볼 수 있음

---
# 2. RoleBinding: 개발자에게 위 역할 부여
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: production
subjects:
- kind: User
  name: developer@example.com
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

**요청 처리 흐름:**

```
개발자가 kubectl get pods -n production 실행:

1. 인증(Authentication): "이 사용자가 누구야?"
   └── 인증서, OIDC 토큰, IAM 등으로 확인

2. 인가(Authorization): "이 사용자가 이 작업을 할 수 있어?"
   └── RBAC 정책 확인
   └── developer@example.com → pod-reader Role → pods get 허용 ✓

3. 결과 반환
```

---

## 네트워크 및 보안

### Network Policy는 Pod간 통신만 제어하나요?

Network Policy는 **Pod를 기준으로** 트래픽을 제어합니다.

```
Network Policy가 제어하는 것:

Ingress (들어오는 트래픽):
├── 다른 Pod → 이 Pod ✓
├── 외부 → 이 Pod ✓ (ipBlock으로)
└── Service → 이 Pod (Service 뒤의 Pod에 적용)

Egress (나가는 트래픽):
├── 이 Pod → 다른 Pod ✓
├── 이 Pod → 외부 ✓ (ipBlock으로)
└── 이 Pod → DNS ✓
```

**실제 예시:**

```yaml
# Backend Pod: Frontend에서만 접근 가능, 외부로는 DB만 접근 가능
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  - Egress
  
  ingress:
  - from:
    # 같은 네임스페이스의 frontend Pod에서만
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
      
  egress:
  # 1. DNS 허용 (필수)
  - to:
    - namespaceSelector: {}
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
  # 2. Database Pod로만
  - to:
    - podSelector:
        matchLabels:
          app: database
    ports:
    - protocol: TCP
      port: 5432
  # 3. 특정 외부 IP로만 (예: 외부 API)
  - to:
    - ipBlock:
        cidr: 203.0.113.0/24
    ports:
    - protocol: TCP
      port: 443
```

---

### mTLS는 암호화에 대한 얘기인가요? Service Mesh가 필요한 이유는?

**mTLS(mutual TLS)란:**

일반 TLS는 클라이언트가 서버를 검증합니다(웹사이트 접속 시 인증서 확인). mTLS는 **양방향**으로 검증합니다.

```
일반 TLS:
클라이언트 ──────────────────► 서버
           "너 진짜 서버 맞아?"
           서버 인증서 확인 ✓

mTLS:
클라이언트 ◄────────────────► 서버
           "너 진짜 서버 맞아?"  "너 진짜 허용된 클라이언트야?"
           서버 인증서 확인 ✓    클라이언트 인증서 확인 ✓
```

**Service Mesh가 필요한 이유:**

Service Mesh 없이 mTLS를 구현하려면:

```
모든 애플리케이션에서 직접 구현:

각 마이크로서비스마다:
├── 인증서 관리 코드 추가
├── TLS 핸드셰이크 구현
├── 인증서 로테이션 구현
├── 인증서 발급/갱신 자동화
└── 각 언어(Java, Go, Python...)별로 구현

문제점:
├── 개발자 부담 증가
├── 언어/프레임워크마다 다르게 구현
├── 일관성 없는 보안 수준
└── 운영 복잡도 폭발
```

Service Mesh(Istio, Linkerd, Cilium)를 사용하면:

```
애플리케이션 코드 변경 없이:

Service Mesh가 해주는 것:
├── 자동으로 인증서 발급 및 주입
├── 모든 트래픽 자동 암호화
├── 인증서 자동 로테이션
├── 서비스 간 인증/인가
└── 트래픽 관찰 (누가 누구와 통신하는지)

개발자:
└── 그냥 HTTP로 코드 작성하면 됨
    (Service Mesh가 자동으로 HTTPS로 변환)
```

**Hybrid Nodes와 Anywhere에서의 차이:**

| 기능 | EKS Hybrid Nodes | EKS Anywhere |
|------|------------------|--------------|
| mTLS 옵션 | Istio, Cilium 모두 가능 | Cilium 기본 내장 |
| 설정 방식 | Add-on으로 설치 | 기본 포함 (Cilium) |

---

### CNI가 하는 역할이 무엇인가요?

CNI(Container Network Interface)는 "Pod가 네트워크에 어떻게 연결되는지"를 담당합니다.

**CNI 없이는:**

```
Pod 생성 시:
├── 컨테이너는 만들어졌는데...
├── IP 주소가 없음!
├── 다른 Pod와 통신 불가!
└── 외부와 통신 불가!
```

**CNI가 하는 일:**

```
Pod 생성 시 CNI가 하는 것:

1. Pod에 네트워크 인터페이스 생성
2. IP 주소 할당 (IPAM)
3. 라우팅 테이블 설정
4. 다른 Pod와 통신 가능하도록 네트워크 구성

추가 기능 (CNI에 따라 다름):
├── Network Policy 적용
├── 암호화 (WireGuard)
├── 로드밸런싱
└── 관측성 (어떤 트래픽이 오가는지)
```

---

### VPC CNI, Cilium, Calico는 어떻게 다른가요?

```
┌─────────────────────────────────────────────────────────────────┐
│                     CNI 비교                                    │
│                                                                 │
│  VPC CNI (AWS 전용):                                           │
│  ├── Pod에 실제 VPC IP 할당                                    │
│  ├── AWS 네이티브 통합 (Security Group, VPC Flow Logs)        │
│  ├── 장점: AWS 서비스와 직접 통신, 기존 네트워크 도구 활용    │
│  └── 단점: AWS 전용, IP 소진 문제 가능                        │
│                                                                 │
│  Cilium (eBPF 기반):                                           │
│  ├── eBPF로 커널 레벨에서 네트워킹 처리                        │
│  ├── 매우 높은 성능                                            │
│  ├── L7 Network Policy (HTTP 메서드 레벨 제어)                │
│  ├── Service Mesh 기능 내장 (Sidecar 없이)                    │
│  ├── Hubble로 뛰어난 관측성                                    │
│  └── 장점: 성능, 기능, 관측성 / 단점: 학습 곡선               │
│                                                                 │
│  Calico:                                                        │
│  ├── 가장 오래되고 안정적인 CNI 중 하나                       │
│  ├── BGP 지원으로 기존 네트워크 인프라와 통합 용이            │
│  ├── Network Policy 표준 구현                                  │
│  └── 장점: 안정성, 문서화 / 단점: eBPF 미지원 (구버전)       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**EKS 환경별 권장:**

| 환경 | 권장 CNI | 이유 |
|------|---------|------|
| EKS (Cloud) | VPC CNI | AWS 네이티브 통합 |
| EKS Hybrid Nodes | Cilium 또는 VPC CNI | AWS 공식 지원 |
| EKS Anywhere | Cilium | 기본 내장, 고성능 |

---

### Cilium이 Network Policy와 Service Mesh를 CNI 수준에서 처리한다는게 무슨 의미인가요?

**기존 방식 (iptables 기반):**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  패킷 처리 흐름 (iptables):                                    │
│                                                                 │
│  패킷 도착                                                      │
│      │                                                          │
│      ▼                                                          │
│  ┌────────────────────────────────────────┐                    │
│  │  iptables rules                         │                    │
│  │  (수천 개의 규칙을 순차적으로 확인)     │  ← 느림!          │
│  └────────────────────────────────────────┘                    │
│      │                                                          │
│      ▼                                                          │
│  ┌────────────────────────────────────────┐                    │
│  │  User Space (Envoy Sidecar)             │  ← 추가 오버헤드  │
│  │  - mTLS 처리                            │                    │
│  │  - L7 정책 적용                         │                    │
│  └────────────────────────────────────────┘                    │
│      │                                                          │
│      ▼                                                          │
│  애플리케이션                                                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Cilium 방식 (eBPF):**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  패킷 처리 흐름 (eBPF):                                        │
│                                                                 │
│  패킷 도착                                                      │
│      │                                                          │
│      ▼                                                          │
│  ┌────────────────────────────────────────┐                    │
│  │  Linux Kernel                           │                    │
│  │  ┌──────────────────────────────────┐  │                    │
│  │  │  eBPF 프로그램                    │  │  ← 커널 레벨     │
│  │  │  - Network Policy 적용           │  │     에서 모두    │
│  │  │  - mTLS (WireGuard)              │  │     처리!        │
│  │  │  - L7 필터링                     │  │                    │
│  │  │  - 관측성 데이터 수집            │  │                    │
│  │  └──────────────────────────────────┘  │                    │
│  └────────────────────────────────────────┘                    │
│      │                                                          │
│      ▼                                                          │
│  애플리케이션 (Sidecar 없음!)                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

성능 차이:
├── iptables: 규칙 수에 비례하여 느려짐
└── eBPF: 규칙 수에 관계없이 일정한 성능
```

---

## 운영 및 관리 도구

### ArgoCD와 Flux는 무엇이 다른가요?

둘 다 GitOps 도구지만 철학과 구현 방식이 다릅니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  ArgoCD:                                                        │
│  ├── 특징:                                                     │
│  │   ├── 풍부한 Web UI                                        │
│  │   ├── 중앙 집중식 관리 (한 곳에서 여러 클러스터 관리)     │
│  │   ├── 배포 승인 워크플로우 지원                            │
│  │   └── ApplicationSet으로 멀티 클러스터 배포               │
│  │                                                             │
│  │   아키텍처:                                                 │
│  │   ┌────────────────┐                                       │
│  │   │  ArgoCD Server │ ───► Cluster 1                       │
│  │   │  (중앙)        │ ───► Cluster 2                       │
│  │   │                │ ───► Cluster 3                       │
│  │   └────────────────┘                                       │
│  │                                                             │
│  └── 적합: UI 필요, 여러 팀이 사용, 승인 프로세스 필요       │
│                                                                 │
│  Flux:                                                          │
│  ├── 특징:                                                     │
│  │   ├── Web UI 없음 (CLI 중심)                               │
│  │   ├── 분산형 (각 클러스터에서 독립 실행)                  │
│  │   ├── 가벼움, 리소스 적게 사용                            │
│  │   └── EKS Anywhere에 기본 내장                            │
│  │                                                             │
│  │   아키텍처:                                                 │
│  │   ┌────────────┐  ┌────────────┐  ┌────────────┐          │
│  │   │ Cluster 1  │  │ Cluster 2  │  │ Cluster 3  │          │
│  │   │ ┌────────┐ │  │ ┌────────┐ │  │ ┌────────┐ │          │
│  │   │ │ Flux   │ │  │ │ Flux   │ │  │ │ Flux   │ │          │
│  │   │ └────────┘ │  │ └────────┘ │  │ └────────┘ │          │
│  │   └────────────┘  └────────────┘  └────────────┘          │
│  │         ▲              ▲              ▲                    │
│  │         └──────────────┴──────────────┘                    │
│  │                    Git Repository                           │
│  │                                                             │
│  └── 적합: CLI 선호, 리소스 제약, EKS Anywhere 사용          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

### EKS Connector를 설치하지 않으면 EKS 대시보드를 활용할 수 없나요?

**EKS Connector의 역할:**

```
EKS Connector는 "외부 클러스터"를 AWS EKS 콘솔에서 
볼 수 있게 해주는 에이전트입니다.

┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  EKS Connector 없이:                                           │
│                                                                 │
│  AWS Console                    EKS Anywhere Cluster           │
│  ┌────────────────┐            ┌────────────────────┐         │
│  │                │     X      │                    │         │
│  │  EKS 클러스터  │ ◄────────►│  (보이지 않음)     │         │
│  │  목록에 없음   │            │                    │         │
│  └────────────────┘            └────────────────────┘         │
│                                                                 │
│  EKS Connector 설치 후:                                        │
│                                                                 │
│  AWS Console                    EKS Anywhere Cluster           │
│  ┌────────────────┐            ┌────────────────────┐         │
│  │                │            │  ┌──────────────┐  │         │
│  │  클러스터 목록 │ ◄──────────│  │ EKS          │  │         │
│  │  에서 확인     │   읽기     │  │ Connector    │  │         │
│  │  가능!         │   전용     │  └──────────────┘  │         │
│  └────────────────┘            └────────────────────┘         │
│                                                                 │
│  주의: Connector는 "보기만" 가능, 제어는 클러스터에서 직접   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Hybrid Nodes의 경우:**

Hybrid Nodes는 EKS 클러스터의 일부이므로 Connector가 필요 없습니다. AWS 콘솔에서 기본으로 보입니다.

```
EKS Hybrid Nodes:
├── AWS 콘솔에서 클러스터 상태 확인 ✓
├── 노드 목록 보기 ✓ (Hybrid Node도 표시)
├── Pod 목록 보기 ✓
├── CloudWatch 메트릭 ✓
└── Control Plane 업그레이드 ✓

EKS Anywhere (Connector 없이):
├── AWS 콘솔에서 보이지 않음
├── 클러스터 관리는 kubectl로
└── 모니터링은 자체 구성 (Prometheus 등)

EKS Anywhere (Connector 있으면):
├── AWS 콘솔에서 클러스터 상태 확인 ✓
├── 노드/Pod 목록 보기 ✓
├── 단, 제어는 여전히 로컬에서
└── "읽기 전용" 가시성
```

---

### nodeadm은 어떤 도구이고, 어떻게 동작하나요?

nodeadm은 EKS Hybrid Nodes를 위한 노드 관리 도구입니다.

**nodeadm이 없었을 때 (수동으로 해야 했던 것들):**

```
온프레미스 서버를 K8s 노드로 만들려면:

1. containerd 설치 및 설정
   $ apt-get install containerd
   $ vi /etc/containerd/config.toml  # 복잡한 설정...

2. kubelet 설치
   $ apt-get install kubelet
   
3. kubelet 설정 파일 작성
   $ vi /var/lib/kubelet/config.yaml
   # 클러스터 정보, 인증 정보 등 수동 입력
   
4. 인증서/kubeconfig 설정
   $ vi /etc/kubernetes/kubelet.conf
   # CA 인증서, 클러스터 엔드포인트 등
   
5. 부트스트랩 토큰 또는 인증서 설정
   # kubeadm join을 위한 토큰 또는 TLS 부트스트래핑
   
6. CNI 플러그인 설치
   $ mkdir -p /opt/cni/bin
   # CNI 바이너리 다운로드 및 설정
   
7. 클러스터에 조인
   $ kubeadm join ...

각 단계마다 에러 가능성, 디버깅 필요...
```

**nodeadm 사용 시:**

```yaml
# nodeConfig.yaml 작성
apiVersion: node.eks.aws/v1alpha1
kind: NodeConfig
spec:
  cluster:
    name: my-cluster
    region: ap-northeast-2
    apiServerEndpoint: https://xxx.eks.amazonaws.com
    certificateAuthority: LS0tLS1CRUdJTi...
    cidr: 10.100.0.0/16
  hybrid:
    ssm:
      activationId: "xxx"
      activationCode: "xxx"
```

```bash
# 단 두 명령으로 끝!
sudo nodeadm init -c file://nodeConfig.yaml

# 끝! nodeadm이 알아서:
# ✓ containerd 설치 및 설정
# ✓ kubelet 설치 및 설정
# ✓ CNI 설치
# ✓ SSM/IAM RA로 인증
# ✓ 클러스터에 조인
```

**nodeadm 주요 명령어:**

```bash
# 노드 초기화 (클러스터에 조인)
sudo nodeadm init -c file://nodeConfig.yaml

# 노드 상태 확인
sudo nodeadm status

# 노드 업그레이드
sudo nodeadm upgrade

# 노드 제거 (클러스터에서 탈퇴)
sudo nodeadm uninstall
```

---

### EKS Anywhere에서 etcd 백업은 어떻게 하나요?

EKS Anywhere에서는 etcd를 직접 관리해야 합니다.

```bash
# 방법 1: etcdctl로 수동 백업
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-$(date +%Y%m%d).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# 백업 검증
ETCDCTL_API=3 etcdctl snapshot status /backup/etcd-20240201.db

# 방법 2: CronJob으로 자동화
```

```yaml
# etcd-backup-cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: etcd-backup
  namespace: kube-system
spec:
  schedule: "0 */6 * * *"  # 6시간마다
  jobTemplate:
    spec:
      template:
        spec:
          hostNetwork: true
          nodeSelector:
            node-role.kubernetes.io/control-plane: ""
          containers:
          - name: backup
            image: bitnami/etcd:latest
            command:
            - /bin/sh
            - -c
            - |
              etcdctl snapshot save /backup/etcd-$(date +%Y%m%d-%H%M).db \
                --endpoints=https://127.0.0.1:2379 \
                --cacert=/etc/kubernetes/pki/etcd/ca.crt \
                --cert=/etc/kubernetes/pki/etcd/server.crt \
                --key=/etc/kubernetes/pki/etcd/server.key
              
              # 7일 이상 된 백업 삭제
              find /backup -name "etcd-*.db" -mtime +7 -delete
            volumeMounts:
            - name: etcd-certs
              mountPath: /etc/kubernetes/pki/etcd
              readOnly: true
            - name: backup
              mountPath: /backup
          volumes:
          - name: etcd-certs
            hostPath:
              path: /etc/kubernetes/pki/etcd
          - name: backup
            persistentVolumeClaim:
              claimName: etcd-backup-pvc
          restartPolicy: OnFailure
```

---

### Container Insights는 에이전트가 필요없나요?

**Container Insights 구성 요소:**

```
Container Insights는 두 가지로 구성:

1. CloudWatch Agent (메트릭 수집)
   └── DaemonSet으로 각 노드에 배포 필요

2. Fluent Bit (로그 수집)
   └── DaemonSet으로 각 노드에 배포 필요
```

**EKS 환경별 설정:**

```
EKS (Cloud):
├── Add-on으로 쉽게 설치 가능
├── aws eks create-addon --addon-name amazon-cloudwatch-observability
└── IRSA로 권한 자동 설정

EKS Hybrid Nodes:
├── 동일하게 Add-on 설치 가능
├── Hybrid Node에서도 CloudWatch로 메트릭/로그 전송
└── AWS 연결 필요

EKS Anywhere:
├── CloudWatch Agent 수동 설치 필요
├── 또는 Prometheus + Grafana 자체 구성 (더 일반적)
└── AWS 연결 없으면 CloudWatch 사용 불가
```

---

### Velero를 사용한다는 것은 어떻게 사용한다는 건가요?

Velero는 Kubernetes 리소스와 PV(Persistent Volume)를 백업/복구하는 도구입니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  Velero가 백업하는 것:                                         │
│  ├── Kubernetes 리소스 (Deployment, Service, ConfigMap 등)    │
│  ├── Persistent Volume 데이터                                  │
│  └── 네임스페이스 전체 또는 선택적으로                        │
│                                                                 │
│  백업 저장 위치:                                                │
│  ├── AWS S3                                                    │
│  ├── GCP Cloud Storage                                         │
│  ├── Azure Blob                                                │
│  └── MinIO (온프레미스)                                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**실제 사용 예시:**

```bash
# 1. Velero 설치 (S3를 백업 위치로)
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.7.0 \
  --bucket my-backup-bucket \
  --backup-location-config region=ap-northeast-2 \
  --secret-file ./credentials-velero

# 2. 수동 백업 생성
velero backup create my-backup \
  --include-namespaces production

# 3. 백업 상태 확인
velero backup describe my-backup

# 4. 백업에서 복구
velero restore create --from-backup my-backup

# 5. 정기 백업 스케줄 생성
velero schedule create daily-backup \
  --schedule="0 2 * * *" \
  --include-namespaces production \
  --ttl 168h  # 7일 보관
```

**재해 복구 시나리오:**

```
상황: production 네임스페이스가 실수로 삭제됨

복구 절차:
$ velero backup get
NAME          STATUS      CREATED
daily-backup  Completed   2024-02-01 02:00:00

$ velero restore create --from-backup daily-backup
Restore request "daily-backup-20240201" submitted successfully.

$ velero restore describe daily-backup-20240201
Phase: Completed
...

$ kubectl get pods -n production
# 복구 완료!
```

---

### Secret 암호화 키는 어떻게 관리해야 하나요?

**문제점:**

```
Kubernetes Secret = base64 인코딩 (암호화가 아님!)

$ echo "password123" | base64
cGFzc3dvcmQxMjM=

$ echo "cGFzc3dvcmQxMjM=" | base64 -d
password123    # 누구나 디코딩 가능!
```

**암호화 계층:**

```
┌─────────────────────────────────────────────────────────────────┐
│                 Secret 보안 계층                                │
│                                                                 │
│  Level 1: etcd 암호화 (저장 시 암호화)                         │
│  ├── EKS/Hybrid Nodes: AWS가 자동 관리                        │
│  └── EKS Anywhere: 직접 구성 필요                             │
│                                                                 │
│  Level 2: 외부 시크릿 관리자                                   │
│  ├── AWS Secrets Manager                                       │
│  ├── HashiCorp Vault                                           │
│  └── 실제 비밀값은 외부에 저장, K8s에는 참조만               │
│                                                                 │
│  Level 3: Sealed Secrets / SOPS                                │
│  └── Git에 저장해도 안전하도록 암호화                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**AWS Secrets Manager 연동 (External Secrets Operator):**

```yaml
# 1. SecretStore 정의 (AWS Secrets Manager 연결)
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secrets
  namespace: production
spec:
  provider:
    aws:
      service: SecretsManager
      region: ap-northeast-2
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets-sa

---
# 2. ExternalSecret 정의 (실제 비밀값 참조)
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: database-credentials
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets
    kind: SecretStore
  target:
    name: db-secret  # 생성될 K8s Secret 이름
  data:
  - secretKey: password
    remoteRef:
      key: prod/database/password  # AWS SM의 키
```

**비용:**

| 서비스 | 비용 |
|--------|------|
| AWS Secrets Manager | $0.40/secret/월 + $0.05/10,000 API 호출 |
| HashiCorp Vault (OSS) | 무료 (인프라 비용만) |
| HashiCorp Vault (Enterprise) | 유료 라이선스 |

---

### 감사 로그(Audit Log)는 어디에 저장되나요?

**EKS / Hybrid Nodes:**

```
기본 설정:
├── Control Plane 감사 로그 → CloudWatch Logs
├── 로그 그룹: /aws/eks/<cluster-name>/cluster
└── 자동 수집, 별도 설정 불필요

활용:
├── CloudWatch Logs Insights로 쿼리
├── 예: "누가 Secret을 읽었는지"
└── 예: "특정 시간에 어떤 API 호출이 있었는지"
```

**쿼리 예시:**

```
# CloudWatch Logs Insights
fields @timestamp, user.username, verb, objectRef.resource, objectRef.name
| filter verb = "delete"
| sort @timestamp desc
| limit 100
```

**EKS Anywhere:**

```
직접 구성 필요:

1. API Server 감사 정책 설정
   /etc/kubernetes/audit-policy.yaml

2. 로그 저장 위치 지정
   --audit-log-path=/var/log/kubernetes/audit.log

3. 로그 수집 (선택)
   ├── Fluent Bit로 수집
   ├── 외부 SIEM으로 전송
   └── ELK Stack으로 분석
```

---

## 기타

### kube-bench는 주기적으로 동작하나요?

kube-bench는 **일회성 도구**입니다. 주기적 검사가 필요하면 별도로 자동화해야 합니다.

```bash
# 수동 실행
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml
kubectl logs job/kube-bench

# 결과 예시
[INFO] 1 Control Plane Security Configuration
[PASS] 1.1.1 Ensure that the API server pod specification file permissions are set to 644
[FAIL] 1.1.2 Ensure that the API server pod specification file ownership is set to root:root
...
```

**주기적 검사 자동화:**

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: kube-bench-scheduled
spec:
  schedule: "0 0 * * 0"  # 매주 일요일
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: kube-bench
            image: aquasec/kube-bench:latest
            command: ["kube-bench"]
          restartPolicy: Never
```

---

### BGP가 어떤 역할을 하나요?

BGP(Border Gateway Protocol)는 "라우터들이 서로 경로 정보를 교환하는 프로토콜"입니다.

**BGP 없이 온프레미스 LoadBalancer:**

```
문제: LoadBalancer 타입 Service의 External IP를
      온프레미스 네트워크에서 어떻게 접근하게 할까?

방법 1 (MetalLB L2 모드):
├── ARP로 IP 광고
├── 단순하지만 단일 노드에서만 처리 (HA 제한)
└── 같은 L2 네트워크 내에서만 동작
```

**BGP 사용 시:**

```
방법 2 (MetalLB BGP 모드):
├── BGP로 External IP 경로를 네트워크 라우터에 광고
├── 라우터가 해당 IP로 오는 트래픽을 클러스터로 전달
├── 여러 노드로 트래픽 분산 가능 (ECMP)
└── 기존 네트워크 인프라와 통합
```

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  BGP로 External IP 광고:                                       │
│                                                                 │
│  K8s Cluster                        네트워크 라우터            │
│  ┌─────────────────────┐           ┌────────────────┐         │
│  │ Service: 10.0.1.100 │   BGP     │                │         │
│  │                     │ ────────► │ "10.0.1.100은  │         │
│  │ [Node1] [Node2]     │           │  저 클러스터로" │         │
│  └─────────────────────┘           └───────┬────────┘         │
│                                            │                   │
│                                            ▼                   │
│                                    외부 트래픽이               │
│                                    올바르게 라우팅             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

### OPA/Gatekeeper나 Falco 같은 보안 도구는 Hybrid Nodes나 Anywhere에서도 적용 가능한가요?

**예, 모두 적용 가능합니다.** 이들은 Kubernetes API를 통해 동작하므로 어떤 K8s 환경에서든 사용할 수 있습니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                 보안 도구 적용 가능 여부                        │
│                                                                 │
│  도구           역할               EKS    Hybrid   Anywhere    │
│  ──────────────────────────────────────────────────────────────│
│  OPA/Gatekeeper 정책 적용         ✓      ✓        ✓           │
│                 (잘못된 설정 차단)                             │
│                                                                 │
│  Falco          런타임 보안       ✓      ✓        ✓           │
│                 (이상 행위 탐지)                               │
│                                                                 │
│  Trivy          이미지 스캐닝     ✓      ✓        ✓           │
│                                                                 │
│  kube-bench     CIS 벤치마크      ✓      ✓        ✓           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**적용 대상:**

```
OPA/Gatekeeper:
└── API Server의 Admission Controller로 동작
    └── Pod, Deployment 등 생성 시 정책 검사

Falco:
└── 각 Worker Node에 DaemonSet으로 배포
    └── 커널 이벤트를 모니터링하여 이상 행위 탐지
```

---

### Amazon Managed Prometheus(AMP)는 기본 활성화되어 있나요?

**아니요, 별도로 설정해야 합니다.**

```
EKS에서 AMP 사용:

1. AMP 워크스페이스 생성 (AWS 콘솔 또는 CLI)
2. ADOT(AWS Distro for OpenTelemetry) 또는 
   Prometheus Agent를 클러스터에 배포
3. 메트릭이 AMP로 전송됨

EKS Anywhere에서 AMP 사용:
├── AWS 연결이 있어야 함
├── Prometheus Agent 직접 구성
└── 또는 자체 Prometheus 운영이 더 일반적
```

**EKS Anywhere에서 Prometheus:**

```
일반적인 구성:

┌─────────────────────────────────────────────────────────────────┐
│                 EKS Anywhere Cluster                            │
│                                                                 │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  Monitoring Namespace                                   │    │
│  │                                                         │    │
│  │  ┌──────────────┐  ┌──────────────┐  ┌─────────────┐  │    │
│  │  │ Prometheus   │  │ Grafana      │  │ AlertManager│  │    │
│  │  │ Server       │  │              │  │             │  │    │
│  │  │ (Worker에    │  │ (Worker에    │  │ (Worker에   │  │    │
│  │  │  Pod로 배포) │  │  Pod로 배포) │  │  Pod로 배포)│  │    │
│  │  └──────────────┘  └──────────────┘  └─────────────┘  │    │
│  │                                                         │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
│  Worker Node 1        Worker Node 2        Worker Node 3       │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐     │
│  │ [App Pods]   │    │ [App Pods]   │    │ [Prometheus] │     │
│  │ [Node Exp.]  │    │ [Node Exp.]  │    │ [Grafana]    │     │
│  └──────────────┘    └──────────────┘    └──────────────┘     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

네, Worker Node에 Pod로 배포합니다.
(Control Plane 노드에는 보통 워크로드를 배포하지 않음)
```

---

[메인으로 돌아가기 →](../README.md)
