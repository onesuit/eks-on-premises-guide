# EKS Hybrid Nodes 시작하기 체크리스트

> EKS Hybrid Nodes를 시작하기 위한 단계별 체크리스트입니다. 각 단계를 순서대로 진행하면서 체크해보세요.

---

## Phase 1: 사전 준비

### AWS 계정 및 권한

```
□ AWS 계정 활성화
□ IAM 사용자/역할 생성
  └── EKS 관리 권한 필요
□ AWS CLI 설치 및 구성
  $ aws configure
  $ aws sts get-caller-identity  # 확인
□ eksctl 설치
  $ curl -sL "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
  $ sudo mv /tmp/eksctl /usr/local/bin
□ kubectl 설치
  $ curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
  $ sudo install kubectl /usr/local/bin/kubectl
```

### 네트워크 연결

```
□ AWS 연결 방식 결정
  ├── AWS Site-to-Site VPN
  └── AWS Direct Connect
□ 네트워크 연결 테스트
  └── AWS 리전의 EKS 엔드포인트에 접근 가능한지 확인
□ 필요한 방화벽 포트 오픈
```

**필수 방화벽 규칙:**

| 방향 | 출발지 | 도착지 | 포트 | 설명 |
|------|--------|--------|------|------|
| Outbound | 온프레미스 노드 | AWS EKS API | 443 | Kubernetes API |
| Outbound | 온프레미스 노드 | AWS SSM | 443 | SSM 통신 (SSM 사용 시) |
| Outbound | 온프레미스 노드 | AWS STS | 443 | IAM 인증 |
| Outbound | 온프레미스 노드 | S3 | 443 | 컨테이너 이미지 (ECR) |
| Inbound | VPC Pod CIDR | 온프레미스 노드 | 10250 | kubelet API |
| Both | 온프레미스 노드들 | 온프레미스 노드들 | All | 클러스터 내 통신 |

### 온프레미스 서버 준비

```
□ 서버 사양 확인
  ├── 최소: 2 vCPU, 4GB RAM
  └── 권장: 4 vCPU, 8GB RAM 이상
□ 지원 OS 확인
  ├── Amazon Linux 2023
  ├── Ubuntu 20.04/22.04
  └── RHEL 8/9
□ 시간 동기화 (NTP) 설정
  $ timedatectl status
  $ sudo systemctl enable --now chronyd
□ 호스트명 설정
  $ sudo hostnamectl set-hostname hybrid-node-01
```

---

## Phase 2: EKS 클러스터 생성

### 클러스터 생성 (AWS Console 또는 eksctl)

**방법 1: AWS Console**

```
□ EKS 콘솔 접속
□ "클러스터 생성" 클릭
□ 클러스터 이름 입력
□ Kubernetes 버전 선택 (최신 안정 버전 권장)
□ 클러스터 서비스 역할 선택/생성
□ VPC 및 서브넷 선택
□ 클러스터 엔드포인트 접근 설정
  └── "Public and Private" 또는 "Private" 선택
□ 클러스터 생성 완료 대기 (~15분)
```

**방법 2: eksctl**

```yaml
# cluster-config.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: my-hybrid-cluster
  region: ap-northeast-2
  version: "1.29"

vpc:
  id: vpc-xxxxxxxxx
  subnets:
    private:
      ap-northeast-2a: { id: subnet-xxxxxx }
      ap-northeast-2b: { id: subnet-yyyyyy }

# Hybrid Nodes 활성화
hybridNetwork:
  remoteNodeNetworks:
    - cidrs:
        - "10.0.0.0/16"  # 온프레미스 노드 CIDR
  remotePodNetworks:
    - cidrs:
        - "10.244.0.0/16"  # 온프레미스 Pod CIDR
```

```bash
□ 설정 파일 검증
$ eksctl create cluster -f cluster-config.yaml --dry-run

□ 클러스터 생성
$ eksctl create cluster -f cluster-config.yaml

□ kubeconfig 업데이트
$ aws eks update-kubeconfig --name my-hybrid-cluster --region ap-northeast-2

□ 클러스터 접근 확인
$ kubectl cluster-info
$ kubectl get nodes
```

---

## Phase 3: 노드 인증 설정

### 방법 A: SSM Hybrid Activations (권장 - 간단함)

```
□ SSM Hybrid Activation 생성
$ aws ssm create-activation \
  --default-instance-name "hybrid-node" \
  --iam-role "HybridNodeRole" \
  --registration-limit 10 \
  --region ap-northeast-2

# 출력:
# ActivationId: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
# ActivationCode: xxxxxxxxxxxxxxxxx

□ Activation ID와 Code 안전하게 보관
```

**필요한 IAM 역할 (HybridNodeRole):**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ssm.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

### 방법 B: IAM Roles Anywhere (기존 PKI 활용 시)

```
□ CA 인증서 준비 (기존 또는 새로 생성)

□ Trust Anchor 생성
$ aws rolesanywhere create-trust-anchor \
  --name "on-prem-ca" \
  --source "sourceData={x509CertificateData=<CA_CERT_PEM>}" \
  --enabled

□ Profile 생성
$ aws rolesanywhere create-profile \
  --name "hybrid-node-profile" \
  --role-arns "arn:aws:iam::123456789:role/HybridNodeRole" \
  --enabled

□ 각 노드에 인증서 배포
```

---

## Phase 4: 노드 등록 (nodeadm)

### nodeadm 설치

```bash
□ nodeadm 다운로드 (각 노드에서)
$ curl -OL 'https://hybrid-assets.eks.amazonaws.com/releases/latest/bin/linux/amd64/nodeadm'
$ sudo mv nodeadm /usr/local/bin/
$ sudo chmod +x /usr/local/bin/nodeadm
$ nodeadm --version
```

### 노드 설정 파일 작성

**SSM 방식:**

```yaml
# nodeConfig.yaml
apiVersion: node.eks.aws/v1alpha1
kind: NodeConfig
spec:
  cluster:
    name: my-hybrid-cluster
    region: ap-northeast-2
  hybrid:
    ssm:
      activationId: "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
      activationCode: "xxxxxxxxxxxxxxxxx"
```

**IAM Roles Anywhere 방식:**

```yaml
# nodeConfig.yaml
apiVersion: node.eks.aws/v1alpha1
kind: NodeConfig
spec:
  cluster:
    name: my-hybrid-cluster
    region: ap-northeast-2
  hybrid:
    iamRolesAnywhere:
      trustAnchorArn: "arn:aws:rolesanywhere:ap-northeast-2:123456789:trust-anchor/xxx"
      profileArn: "arn:aws:rolesanywhere:ap-northeast-2:123456789:profile/xxx"
      roleArn: "arn:aws:iam::123456789:role/HybridNodeRole"
      certificatePath: "/etc/hybrid/certs/node.crt"
      privateKeyPath: "/etc/hybrid/certs/node.key"
```

### 노드 초기화

```bash
□ 설정 파일 검증
$ sudo nodeadm init --config-source file://nodeConfig.yaml --dry-run

□ 노드 초기화 실행
$ sudo nodeadm init --config-source file://nodeConfig.yaml

□ 노드 상태 확인
$ sudo nodeadm status

□ 클러스터에서 노드 확인
$ kubectl get nodes
NAME                STATUS   ROLES    AGE   VERSION
hybrid-node-01      Ready    <none>   1m    v1.29.x
```

---

## Phase 5: CNI 설정

### VPC CNI (AWS와 통합, Pod에 VPC IP 할당)

```bash
□ VPC CNI 설치 확인
$ kubectl get pods -n kube-system -l k8s-app=aws-node

□ Hybrid Node용 설정 (필요시)
# 온프레미스 Pod에 별도 CIDR 사용
```

### Cilium (권장 - 고성능, 다기능)

```bash
□ Cilium CLI 설치
$ curl -L --remote-name-all https://github.com/cilium/cilium-cli/releases/latest/download/cilium-linux-amd64.tar.gz
$ sudo tar xzvfC cilium-linux-amd64.tar.gz /usr/local/bin

□ Cilium 설치
$ cilium install --version 1.15.0

□ 설치 확인
$ cilium status
$ kubectl get pods -n kube-system -l app.kubernetes.io/part-of=cilium
```

---

## Phase 6: 검증

### 기본 기능 테스트

```bash
□ 테스트 Pod 배포
$ kubectl run test-pod --image=nginx --restart=Never
$ kubectl get pod test-pod -o wide

□ Pod 네트워크 테스트
$ kubectl exec -it test-pod -- curl -I localhost

□ 노드 간 통신 테스트
$ kubectl run test-pod-2 --image=busybox --restart=Never -- sleep 3600
$ kubectl exec -it test-pod-2 -- ping <test-pod-ip>

□ DNS 테스트
$ kubectl exec -it test-pod -- nslookup kubernetes.default

□ 테스트 리소스 정리
$ kubectl delete pod test-pod test-pod-2
```

### AWS 서비스 통합 테스트

```bash
□ CloudWatch 로그 확인
# AWS Console > CloudWatch > Log groups > /aws/eks/<cluster>/cluster

□ IAM 인증 테스트
$ kubectl auth can-i get pods --as system:node:hybrid-node-01
```

---

## Phase 7: 모니터링 설정

### Container Insights (권장)

```bash
□ CloudWatch Observability Add-on 설치
$ aws eks create-addon \
  --cluster-name my-hybrid-cluster \
  --addon-name amazon-cloudwatch-observability

□ 메트릭 확인
# AWS Console > CloudWatch > Container Insights
```

### Prometheus + Grafana (대안)

```bash
□ kube-prometheus-stack 설치
$ helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
$ helm install monitoring prometheus-community/kube-prometheus-stack -n monitoring --create-namespace

□ Grafana 접근
$ kubectl port-forward svc/monitoring-grafana 3000:80 -n monitoring
# 브라우저에서 http://localhost:3000 접속
# 기본 계정: admin / prom-operator
```

---

## 체크리스트 요약

```
Pre-requisites:
□ AWS 계정 및 권한 설정
□ 네트워크 연결 (VPN/Direct Connect)
□ 방화벽 규칙 설정
□ 온프레미스 서버 준비

Cluster:
□ EKS 클러스터 생성
□ Hybrid Network 설정 (CIDR)

Authentication:
□ SSM Hybrid Activation 또는 IAM Roles Anywhere 설정

Nodes:
□ nodeadm 설치
□ 노드 초기화 및 등록

Networking:
□ CNI 설정 (VPC CNI 또는 Cilium)

Verification:
□ 노드 상태 확인
□ Pod 배포 테스트
□ 네트워크 테스트

Monitoring:
□ Container Insights 또는 Prometheus 설정
```

---

## 트러블슈팅

### 노드가 Not Ready 상태

```bash
# 노드 상태 확인
$ kubectl describe node <node-name>

# nodeadm 로그 확인
$ journalctl -u nodeadm -f

# kubelet 로그 확인  
$ journalctl -u kubelet -f

# 일반적인 원인:
# 1. 네트워크 연결 문제 (방화벽)
# 2. SSM/IAM RA 인증 실패
# 3. CNI 미설정
```

### Pod가 Pending 상태

```bash
# Pod 이벤트 확인
$ kubectl describe pod <pod-name>

# 일반적인 원인:
# 1. 리소스 부족 (CPU, 메모리)
# 2. 노드 셀렉터 불일치
# 3. Taints/Tolerations 문제
```

### AWS 서비스 접근 불가

```bash
# IAM 역할 확인
$ aws sts get-caller-identity

# 네트워크 연결 확인
$ curl -I https://eks.ap-northeast-2.amazonaws.com

# 일반적인 원인:
# 1. VPN/Direct Connect 연결 문제
# 2. IAM 역할 권한 부족
# 3. Security Group/방화벽 설정
```

---

> ✅ **완료**
>
> **축하합니다!** 체크리스트를 모두 완료했다면 EKS Hybrid Nodes 환경이 준비된 것입니다. [Best Practices](best-practices.md)를 참고하여 프로덕션 환경을 더 안정적으로 운영하세요.
