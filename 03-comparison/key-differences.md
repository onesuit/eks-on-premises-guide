# 핵심 차이점 비교

> "구체적으로 뭐가 다른지 표로 정리해주세요!"

## 선택하기 전에: 실제 도입 사례

**사례 A: 금융회사 — EKS Hybrid Nodes 선택**

이 회사는 AWS에서 이미 여러 서비스를 운영 중이었고, 규제 요건으로 일부 워크로드를 자체 데이터센터에서 실행해야 했습니다. Kubernetes 전담 인력은 1명뿐이었습니다.

EKS Hybrid Nodes를 선택한 이유: Control Plane 관리를 AWS에 맡기고, 기존 IAM과 CloudWatch를 그대로 활용할 수 있었습니다. 온프레미스 서버 3대를 Worker Node로 등록하는 데 반나절이면 충분했고, 운영 방식도 기존 EKS와 동일해서 학습 부담이 없었습니다.

**사례 B: 제조업체 — EKS Anywhere 선택**

이 회사의 공장은 인터넷이 불안정한 지역에 있었고, 네트워크 단절 상황에서도 생산 시스템이 멈추면 안 됐습니다. 또한 보안 정책상 어떤 데이터도 외부 클라우드로 나가면 안 됐습니다.

EKS Anywhere를 선택한 이유: 완전히 격리된 환경에서 독립적으로 운영할 수 있었습니다. Control Plane 관리 부담이 있었지만, Kubernetes 경험이 풍부한 DevOps 팀(3명)이 있어서 감당할 수 있었습니다.

**결국 "어떤 제약이 있고, 어떤 역량이 있는가"에 따라 답이 달라집니다.**

## 한눈에 보는 핵심 차이

두 솔루션의 가장 큰 차이점은 **Control Plane의 위치와 관리 주체**입니다. 이 한 가지 차이가 운영 방식, 비용 구조, 제약 조건 모든 것을 결정합니다.

| 구분         | EKS Hybrid Nodes                                | EKS Anywhere                 |
| ---------- | ----------------------------------------------- | ---------------------------- |
| **한 줄 요약** | AWS가 두뇌(Control Plane)를 관리, 여러분은 손발(Worker)만 제공 | 두뇌와 손발 모두 여러분의 데이터센터에서 직접 운영 |
| **철학**     | "클라우드의 편리함을 온프레미스로"                             | "온프레미스의 독립성을 유지하면서 EKS 경험을"  |

## 결정적 차이: Control Plane의 위치

### EKS Hybrid Nodes — 클라우드에 두뇌, 온프레미스에 손발

EKS Hybrid Nodes에서 **API Server, etcd, Scheduler, Controller Manager**는 모두 AWS 클라우드에서 실행됩니다. 여러분의 온프레미스 서버는 **Worker Node**로만 동작하며, VPN이나 Direct Connect를 통해 클라우드의 Control Plane과 통신합니다.

```
AWS Cloud
┌──────────────────────────────────────────────────┐
│  EKS Control Plane (AWS 관리)                     │
│  ┌──────────┬───────────┬──────────┬───────────┐ │
│  │   etcd   │API Server │Scheduler │Controller │ │
│  └──────────┴───────────┴──────────┴───────────┘ │
└──────────────────────────┬───────────────────────┘
                           │ VPN / Direct Connect
┌──────────────────────────┴───────────────────────┐
│  On-Premises (여러분이 관리)                      │
│  ┌──────────────┬──────────────┬──────────────┐  │
│  │   Worker 1   │   Worker 2   │   Worker 3   │  │
│  │   kubelet    │   kubelet    │   kubelet    │  │
│  │  containerd  │  containerd  │  containerd  │  │
│  └──────────────┴──────────────┴──────────────┘  │
└──────────────────────────────────────────────────┘
```

**이 구조가 의미하는 것:**

* ✅ **Control Plane 운영 부담 제로** — 업그레이드, 패치, 고가용성 모두 AWS가 담당
* ✅ **AWS 서비스 네이티브 통합** — IAM, CloudWatch, Secrets Manager 등 즉시 사용
* ⚠️ **AWS 연결 필수** — 네트워크 단절 시 새 Pod 스케줄링 불가
* ⚠️ **레이턴시 고려** — API Server가 클라우드에 있으므로 kubectl 응답 시간 증가

### EKS Anywhere — 모든 것이 온프레미스에

EKS Anywhere는 **Control Plane과 Worker Node 모두** 온프레미스에서 실행됩니다. AWS 연결은 선택 사항이며, 완전히 격리된 환경에서도 운영할 수 있습니다.

```
On-Premises (전체 - 여러분이 관리)
┌──────────────────────────────────────────────────┐
│  Control Plane Nodes                              │
│  ┌─────────────┬─────────────┬─────────────┐     │
│  │   Master1   │   Master2   │   Master3   │     │
│  │   etcd      │   etcd      │   etcd      │     │
│  │  API Server │  API Server │  API Server │     │
│  └─────────────┴─────────────┴─────────────┘     │
│                        │                          │
│  ┌─────────────┬───────┴─────┬─────────────┐     │
│  │   Worker 1  │   Worker 2  │   Worker 3  │     │
│  └─────────────┴─────────────┴─────────────┘     │
└──────────────────────────────────────────────────┘
           │
           │ (선택적 연결)
           ▼
      AWS Cloud (콘솔 통합용)
```

**이 구조가 의미하는 것:**

* ✅ **완전한 독립성** — 인터넷 없이도 클러스터 운영 가능 (Air-gapped)
* ✅ **완전한 통제권** — 모든 구성요소에 대한 직접 접근
* ✅ **기본 라이선스 비용 없음** — 하드웨어와 운영 인력 비용만 발생
* ⚠️ **운영 책임 증가** — Control Plane HA, 업그레이드, 백업 모두 직접 관리
* ⚠️ **Kubernetes 전문성 필요** — etcd 관리, 인증서 갱신 등 고급 운영 역량 필요

## 상세 비교: 어떤 상황에서 무엇이 유리한가

### 1. 기본 특성

| 항목                | EKS Hybrid Nodes | EKS Anywhere | 어떤 상황에서 중요한가        |
| ----------------- | ---------------- | ------------ | ------------------- |
| **출시일**           | 2024년 12월        | 2021년 9월     | EKS Anywhere가 더 성숙함 |
| **Control Plane** | AWS 관리           | 직접 관리        | **운영 역량에 따라 결정적**   |
| **AWS 연결**        | 필수               | 선택           | 규제/보안 요건에 따라 결정적    |
| **Air-gapped**    | ❌ 불가             | ✅ 가능         | 군, 금융, 의료 등에서 필수    |

> 🎯 **핵심 판단**: 네트워크 격리가 필수인가요? 그렇다면 **EKS Anywhere**가 유일한 선택입니다.

### 2. 인프라 및 OS 지원 비교

#### EKS Hybrid Nodes

| 인프라                | AL2023 | Bottlerocket | Ubuntu | RHEL |
| ------------------ | :----: | :----------: | :----: | :--: |
| **VMware vSphere** |    ✅   |       ✅      |    ✅   |   ✅  |
| **Bare Metal**     |    ✅   |       ❌      |    ✅   |   ✅  |
| **기타 VM 환경**       |    ✅   |       ❌      |    ✅   |   ✅  |

* **인프라 비의존적**: 대부분의 환경에서 동작
* **OS 혼합 가능**: 클러스터 내 여러 OS 사용 가능
* **Bottlerocket**: VMware vSphere에서만 지원

#### EKS Anywhere

| 인프라                | Bottlerocket | Ubuntu | RHEL |
| ------------------ | :----------: | :----: | :--: |
| **VMware vSphere** |       ✅      |    ✅   |   ✅  |
| **Bare Metal**     |       ✅      |    ✅   |   ✅  |
| **Nutanix**        |       ❌      |    ✅   |   ✅  |
| **CloudStack**     |       ❌      |    ❌   |   ✅  |
| **AWS Snow**       |       ❌      |    ✅   |   ❌  |

* **인프라 프로바이더 통합**: 각 인프라에 맞는 드라이버 제공
* **단일 OS만 지원**: 클러스터 내 OS 혼합 불가
* **OVA 직접 빌드 필요**: Ubuntu/RHEL

#### 핵심 차이점 요약

| 항목                             | EKS Hybrid Nodes | EKS Anywhere |
| ------------------------------ | :--------------: | :----------: |
| **인프라 의존성**                    |     없음 (비의존적)    |  있음 (프로바이더별) |
| **클러스터 내 OS 혼합**               |       ✅ 가능       |   ❌ 단일 OS만   |
| **AL2023 지원**                  |         ✅        |       ❌      |
| **Bottlerocket on Bare Metal** |         ❌        |       ✅      |
| **Air-gapped 운영**              |         ❌        |       ✅      |

### 3. 네트워크

네트워크 구성은 두 솔루션에서 가장 큰 운영 차이를 만듭니다.

| 항목           | EKS Hybrid Nodes  | EKS Anywhere  |
| ------------ | ----------------- | ------------- |
| **AWS 연결**   | VPN/DX **필수**     | 불필요           |
| **Pod CIDR** | VPC와 겹치면 안 됨      | 자유롭게 설정       |
| **기본 CNI**   | VPC CNI 또는 Cilium | Cilium        |
| **로드밸런서**    | AWS ELB 가능        | MetalLB 직접 구성 |

**EKS Hybrid Nodes 네트워크 제약:**

AWS VPC와 온프레미스 Pod 네트워크가 겹치면 라우팅 충돌이 발생합니다. 예를 들어 VPC가 `10.0.0.0/16`이라면, 온프레미스 Pod CIDR은 `172.16.0.0/16` 등 다른 대역을 사용해야 합니다.

```yaml
# EKS Hybrid Nodes - Pod CIDR 주의사항
VPC CIDR: 10.0.0.0/16
On-prem Pod CIDR: 172.16.0.0/16  # 겹치면 안 됨!
On-prem Node CIDR: 192.168.0.0/24
```

> 🎯 **핵심 판단**: 기존 네트워크 CIDR이 복잡하거나 AWS VPC와 겹치나요? **EKS Anywhere**가 더 유연합니다.

### 4. 관리 및 운영

| 항목        | EKS Hybrid Nodes | EKS Anywhere     | 운영 부담             |
| --------- | ---------------- | ---------------- | ----------------- |
| **업그레이드** | AWS 콘솔/CLI       | eksctl anywhere  | Hybrid가 더 쉬움      |
| **노드 등록** | SSM 또는 IAM RA    | EKS-A CLI        | 비슷함               |
| **인증**    | IAM 통합           | OIDC 직접 구성       | **Hybrid가 훨씬 쉬움** |
| **모니터링**  | CloudWatch 통합    | Prometheus 직접 구성 | **Hybrid가 훨씬 쉬움** |
| **로깅**    | CloudWatch Logs  | 직접 구성            | **Hybrid가 훨씬 쉬움** |
| **백업**    | AWS 관리           | Velero 등 직접 구성   | **Hybrid가 훨씬 쉬움** |

> 🎯 **핵심 판단**: DevOps 인력이 제한적인가요? **EKS Hybrid Nodes**가 운영 부담을 크게 줄여줍니다.

### 5. 비용 구조

비용을 단순히 "월 얼마"로 비교하기는 어렵습니다. **숨은 비용**을 포함해서 판단해야 합니다.

| 비용 항목                  | EKS Hybrid Nodes   | EKS Anywhere |
| ---------------------- | ------------------ | ------------ |
| **Control Plane**      | $0.10/시간 (\~$73/월) | $0           |
| **Worker Node**        | vCPU당 $0.01/시간     | $0           |
| **AWS 연결**             | VPN/DX 비용 발생       | 없음           |
| **Enterprise Support** | AWS Support 포함     | 별도 구독 필요     |

**숨은 비용 비교:**

| 숨은 비용                   | EKS Hybrid Nodes | EKS Anywhere  |
| ----------------------- | ---------------- | ------------- |
| **Control Plane 운영 인력** | 불필요              | **필요**        |
| **Kubernetes 전문가**      | 기본 수준            | **고급 수준 필요**  |
| **모니터링 인프라**            | CloudWatch 포함    | **직접 구축**     |
| **장애 복구 시간**            | AWS SLA          | **자체 역량에 의존** |

> 🎯 **핵심 판단**: 단순 비용 = EKS Anywhere 저렴. **TCO(총 소유 비용)** = 상황에 따라 다름.\
> 인력 비용까지 고려하면 EKS Hybrid Nodes가 더 저렴할 수 있습니다.

### 6. 기능 비교

| 기능                  | EKS Hybrid Nodes | EKS Anywhere | 왜 중요한가                    |
| ------------------- | ---------------- | ------------ | ------------------------- |
| **AWS IAM 통합**      | ✅                | ❌            | 기존 AWS 권한 체계 활용           |
| **EKS Add-ons**     | ✅                | 일부 ✅         | CoreDNS, kube-proxy 자동 관리 |
| **Pod Identity**    | ✅                | ❌            | Pod에서 AWS 서비스 접근          |
| **GuardDuty**       | ✅                | ❌            | 보안 위협 탐지                  |
| **Disconnected 운영** | ❌                | ✅            | Air-gapped 환경 필수          |
| **GitOps**          | Flux/ArgoCD 선택   | Flux 기본 포함   | 배포 자동화                    |

> 🎯 **핵심 판단**: AWS 서비스를 많이 사용하나요? **EKS Hybrid Nodes**가 통합이 매끄럽습니다.

## 지원 환경 상세

### EKS Hybrid Nodes

| 구분          | 지원 범위                                                                    |
| ----------- | ------------------------------------------------------------------------ |
| **OS**      | Amazon Linux 2023, Bottlerocket (vSphere만), Ubuntu 20.04/22.04, RHEL 8/9 |
| **아키텍처**    | x86\_64, ARM64                                                           |
| **최소 사양**   | 1 vCPU, 1GB RAM                                                          |
| **필수 구성요소** | containerd, SSM Agent 또는 IAM RA 인증서                                      |
| **네트워크**    | AWS 엔드포인트 접근 가능해야 함                                                      |

### EKS Anywhere

| 구분        | 지원 범위                                                                       |
| --------- | --------------------------------------------------------------------------- |
| **인프라**   | VMware vSphere 7.0+, Bare Metal (Tinkerbell), Nutanix, CloudStack, AWS Snow |
| **OS**    | Bottlerocket (권장), Ubuntu 20.04/22.04, RHEL 8 (일부)                          |
| **구성 옵션** | Standalone, Management + Workload 클러스터                                      |

## 최종 선택 가이드

### 여러분의 상황은 어떤가요?

선택을 더 쉽게 하기 위해 몇 가지 질문에 답해보세요:

**Q1. 인터넷이 끊겨도 클러스터가 작동해야 하나요?**

* "예" → **EKS Anywhere** (유일한 선택)
* "아니오, 항상 연결되어 있음" → 다음 질문으로

**Q2. Kubernetes 운영 전담 인력이 있나요?**

* "없거나 1명 이하" → **EKS Hybrid Nodes** (운영 부담 최소화)
* "2명 이상, 경험 풍부" → 다음 질문으로

**Q3. 기존에 VMware나 Nutanix를 사용 중인가요?**

* "예" → **EKS Anywhere** (기존 인프라 활용)
* "아니오, 베어메탈이나 새 환경" → 다음 질문으로

**Q4. AWS 서비스(IAM, CloudWatch, Secrets Manager)를 많이 사용하나요?**

* "예" → **EKS Hybrid Nodes** (네이티브 통합)
* "아니오, AWS 의존도 낮음" → **EKS Anywhere** (라이선스 비용 절감)

### EKS Hybrid Nodes를 선택하세요, 만약:

* ✅ AWS 클라우드와 긴밀한 통합이 필요하다
* ✅ Kubernetes 운영 경험이 제한적이다
* ✅ 기존 AWS IAM/CloudWatch 인프라를 활용하고 싶다
* ✅ Control Plane 운영에 시간을 쓰고 싶지 않다
* ✅ 안정적인 AWS 네트워크 연결이 가능하다

**주의할 점:**

* AWS 연결이 끊기면 새 Pod 배포 불가
* vCPU 기반 추가 비용 발생
* VPC CIDR과 Pod CIDR 충돌 피해야 함

### EKS Anywhere를 선택하세요, 만약:

* ✅ 완전한 네트워크 격리(Air-gapped)가 필요하다
* ✅ 규제 요건으로 데이터가 온프레미스에 있어야 한다
* ✅ 기존 VMware/Nutanix 인프라를 활용하려 한다
* ✅ Kubernetes 운영 전문성을 보유하고 있다
* ✅ 라이선스 비용을 최소화하고 싶다

**주의할 점:**

* Control Plane HA, 업그레이드, 백업 모두 직접 관리
* etcd 운영, 인증서 갱신 등 고급 역량 필요
* 장애 발생 시 자체 해결 역량 필요

***

**다음으로**: 구체적인 선택 기준이 필요하다면 [선택 가이드](decision-guide.md)를 참고하세요.
