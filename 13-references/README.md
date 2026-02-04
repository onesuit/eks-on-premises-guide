# 참고 자료

> EKS Hybrid Nodes와 EKS Anywhere를 학습하고 운영하는 데 도움이 되는 자료들입니다.

---

## 공식 문서

### AWS 공식 문서

| 리소스 | 링크 | 설명 |
|--------|------|------|
| EKS 사용 설명서 | [docs.aws.amazon.com/eks](https://docs.aws.amazon.com/eks/) | EKS 공식 문서 |
| EKS Hybrid Nodes | [EKS Hybrid Nodes Guide](https://docs.aws.amazon.com/eks/latest/userguide/hybrid-nodes-overview.html) | Hybrid Nodes 상세 가이드 |
| EKS Anywhere | [anywhere.eks.amazonaws.com](https://anywhere.eks.amazonaws.com/) | EKS Anywhere 공식 사이트 |
| EKS Best Practices | [aws.github.io/aws-eks-best-practices](https://aws.github.io/aws-eks-best-practices/) | AWS 권장 운영 가이드 |

### Kubernetes 공식 문서

| 리소스 | 링크 | 설명 |
|--------|------|------|
| Kubernetes Docs | [kubernetes.io/docs](https://kubernetes.io/docs/) | Kubernetes 공식 문서 |
| kubectl Reference | [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/) | kubectl 명령어 참고 |
| API Reference | [API Reference](https://kubernetes.io/docs/reference/kubernetes-api/) | Kubernetes API 레퍼런스 |

---

## 학습 자료

### 온라인 강의

| 플랫폼 | 강의 | 수준 |
|--------|------|------|
| Udemy | CKA/CKAD 준비 강의 | 중급 |
| Coursera | Google Cloud의 Kubernetes 강의 | 입문~중급 |
| A Cloud Guru | EKS Deep Dive | 중급~고급 |
| Linux Foundation | Kubernetes Fundamentals (LFS258) | 입문 |

### 자격증

```
권장 학습 순서:

1. Kubernetes 기초
   └── CKA (Certified Kubernetes Administrator)
       ├── 클러스터 관리 역량 검증
       └── 난이도: 중간

2. 애플리케이션 개발
   └── CKAD (Certified Kubernetes Application Developer)
       ├── 애플리케이션 배포 역량
       └── 난이도: 중간

3. 보안 (선택)
   └── CKS (Certified Kubernetes Security Specialist)
       ├── 보안 전문 역량
       └── 난이도: 어려움

4. AWS 전문가 (선택)
   └── AWS SAA/SAP
       ├── AWS 아키텍처 이해
       └── Hybrid 환경 설계
```

---

## 도구 및 유틸리티

### 필수 CLI 도구

| 도구 | 설치 | 용도 |
|------|------|------|
| kubectl | [설치 가이드](https://kubernetes.io/docs/tasks/tools/) | Kubernetes CLI |
| eksctl | [eksctl.io](https://eksctl.io/) | EKS 클러스터 관리 |
| helm | [helm.sh](https://helm.sh/) | 패키지 관리 |
| aws cli | [AWS CLI](https://aws.amazon.com/cli/) | AWS 리소스 관리 |
| cilium cli | [Cilium CLI](https://docs.cilium.io/en/stable/gettingstarted/k8s-install-default/) | Cilium 관리 |

### 유용한 kubectl 플러그인

```bash
# krew 설치 (플러그인 매니저)
(
  set -x; cd "$(mktemp -d)" &&
  OS="$(uname | tr '[:upper:]' '[:lower:]')" &&
  ARCH="$(uname -m | sed -e 's/x86_64/amd64/' -e 's/\(arm\)\(64\)\?.*/\1\2/' -e 's/aarch64$/arm64/')" &&
  KREW="krew-${OS}_${ARCH}" &&
  curl -fsSLO "https://github.com/kubernetes-sigs/krew/releases/latest/download/${KREW}.tar.gz" &&
  tar zxvf "${KREW}.tar.gz" &&
  ./"${KREW}" install krew
)

# 유용한 플러그인들
kubectl krew install ctx        # 컨텍스트 전환
kubectl krew install ns         # 네임스페이스 전환
kubectl krew install neat       # YAML 정리
kubectl krew install tree       # 리소스 계층 구조
kubectl krew install images     # 이미지 목록
kubectl krew install resource-capacity  # 리소스 용량
```

### 개발/디버깅 도구

| 도구 | 용도 | 링크 |
|------|------|------|
| k9s | 터미널 UI | [k9scli.io](https://k9scli.io/) |
| Lens | 데스크톱 UI | [k8slens.dev](https://k8slens.dev/) |
| stern | 멀티 Pod 로그 | [github.com/stern/stern](https://github.com/stern/stern) |
| kubectx/kubens | 컨텍스트/네임스페이스 전환 | [github.com/ahmetb/kubectx](https://github.com/ahmetb/kubectx) |

---

## 커뮤니티

### Slack 채널

- **Kubernetes Slack**: [kubernetes.slack.com](https://slack.k8s.io/)
  - #eks
  - #aws
  - #cilium
- **AWS Community**: AWS re:Post, AWS 한국 사용자 그룹

### GitHub 저장소

| 저장소 | 설명 |
|--------|------|
| [aws/eks-anywhere](https://github.com/aws/eks-anywhere) | EKS Anywhere 소스 |
| [aws/eks-charts](https://github.com/aws/eks-charts) | EKS Helm Charts |
| [cilium/cilium](https://github.com/cilium/cilium) | Cilium CNI |
| [kubernetes/kubernetes](https://github.com/kubernetes/kubernetes) | Kubernetes 소스 |

### 블로그 및 뉴스

- [AWS Blog - Containers](https://aws.amazon.com/blogs/containers/)
- [Kubernetes Blog](https://kubernetes.io/blog/)
- [CNCF Blog](https://www.cncf.io/blog/)
- [The New Stack](https://thenewstack.io/)

---

## 추가 읽을거리

### 책

| 제목 | 저자 | 수준 |
|------|------|------|
| Kubernetes Up & Running | Kelsey Hightower 등 | 입문~중급 |
| Kubernetes Patterns | Bilgin Ibryam | 중급 |
| Production Kubernetes | Josh Rosso 등 | 고급 |
| Cloud Native DevOps with Kubernetes | John Arundel | 중급 |

### 관련 CNCF 프로젝트

```
네트워킹:
├── Cilium (eBPF 기반 CNI)
├── Calico (네트워크 정책)
└── Istio/Linkerd (Service Mesh)

모니터링:
├── Prometheus (메트릭)
├── Grafana (시각화)
├── Jaeger (트레이싱)
└── Loki (로깅)

CI/CD:
├── ArgoCD (GitOps)
├── Flux (GitOps)
└── Tekton (CI/CD 파이프라인)

보안:
├── Falco (런타임 보안)
├── OPA/Gatekeeper (정책)
├── Trivy (이미지 스캐닝)
└── cert-manager (인증서 관리)

스토리지:
├── Rook (Ceph 오케스트레이션)
├── Longhorn (분산 블록 스토리지)
└── MinIO (S3 호환 오브젝트 스토리지)
```

---

## FAQ

자주 묻는 질문과 상세한 답변은 [FAQ 페이지](faq.md)를 참고하세요.

다루는 주제:
- 인증 및 접근 관리 (IAM Roles Anywhere, IRSA, RBAC)
- 네트워크 (CNI, Network Policy, BGP)
- 운영 도구 (ArgoCD vs Flux, Velero, Container Insights)
- 보안 (mTLS, Secret 관리, 정책 엔진)
- 기타 자주 묻는 질문들

---

> 💡 **참고**
>
> **문서 기여**: 이 가이드에 오류가 있거나 개선 제안이 있다면 언제든 알려주세요!
