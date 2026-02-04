# EKS 온프레미스 운영 가이드

> **AWS EKS Hybrid Nodes vs EKS Anywhere: 어떤 것을 선택해야 할까?**

## 이 가이드의 목적

클라우드 네이티브 시대, 많은 기업들이 Kubernetes를 도입하고 있습니다. 하지만 모든 워크로드를 퍼블릭 클라우드에 올릴 수는 없습니다. 규정 준수, 데이터 주권, 레이턴시, 기존 인프라 활용 등 다양한 이유로 **온프레미스에서 Kubernetes를 운영**해야 하는 상황이 발생합니다.

AWS는 이런 요구에 대응하기 위해 두 가지 솔루션을 제공합니다:

| 솔루션 | 한 줄 요약 |
|--------|-----------|
| **EKS Hybrid Nodes** | AWS가 Control Plane을 관리하고, 여러분의 온프레미스 서버를 Worker Node로 연결 |
| **EKS Anywhere** | Control Plane부터 Worker Node까지 모두 온프레미스에서 직접 운영 |

## 누구를 위한 가이드인가요?

- Kubernetes를 처음 접하는 인프라/DevOps 엔지니어
- 온프레미스 EKS 도입을 검토 중인 아키텍트
- 클라우드와 온프레미스를 함께 운영해야 하는 팀

## 이 가이드에서 다루는 내용

```
📚 목차 미리보기

Part 1: 기초 다지기
├── Kubernetes가 뭔가요?
├── Control Plane vs Worker Node
└── 왜 온프레미스에서 K8s를 운영할까요?

Part 2: 솔루션 비교
├── EKS Hybrid Nodes 깊이 알아보기
├── EKS Anywhere 깊이 알아보기
└── 우리 조직에 맞는 선택은?

Part 3: 실전 운영
├── Multi-Cluster 전략
├── 보안 5계층 모델
├── 네트워크 구성
├── Day-2 운영
└── 비용 분석

Part 4: 시작하기
├── 체크리스트
├── Best Practices
└── 참고 자료
```

## 시작하기 전에

> 💡 **참고**
>
> **이 가이드의 특징**
>
> - 🎯 **왜(Why)** 에 집중: 단순 설정 나열이 아닌, 왜 이런 구성이 필요한지 설명
> - 📖 **스토리텔링**: 기술 문서가 아닌 읽을 수 있는 가이드
> - 🔧 **실무 중심**: 실제 운영에서 마주치는 고민과 해결책

> ⚠️ **주의**
>
> **주의사항**
>
> 이 가이드는 2024-2025년 기준으로 작성되었습니다. AWS 서비스는 빠르게 변화하므로, 실제 도입 시에는 [AWS 공식 문서](https://docs.aws.amazon.com/)를 함께 참고하세요.

---

## 빠른 시작

어떤 솔루션이 맞는지 빨리 알고 싶다면?

| 상황 | 추천 |
|------|------|
| AWS 클라우드와 긴밀한 통합이 필요하다 | → [EKS Hybrid Nodes](04-hybrid-nodes/README.md) |
| 완전한 온프레미스 독립 운영이 필요하다 | → [EKS Anywhere](05-anywhere/README.md) |
| 잘 모르겠다, 비교부터 보고 싶다 | → [솔루션 비교](03-comparison/README.md) |

---

**다음 장에서는** Kubernetes가 무엇인지, 왜 컨테이너 오케스트레이션이 필요한지부터 차근차근 알아보겠습니다.

[시작하기 →](01-introduction/README.md)
