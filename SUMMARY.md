# Table of contents

* [EKS 온프레미스 운영 가이드](README.md)

## Part 1: 기초 다지기

* [들어가며](01-introduction/README.md)
  * [왜 온프레미스에서 Kubernetes인가?](01-introduction/why-kubernetes-on-premises.md)
* [Kubernetes 기초](02-kubernetes-basics/README.md)
  * [핵심 개념 이해하기](02-kubernetes-basics/core-concepts.md)
  * [Control Plane 깊이 알아보기](02-kubernetes-basics/control-plane.md)
  * [Worker Node 이해하기](02-kubernetes-basics/worker-nodes.md)

## Part 2: 솔루션 비교

* [EKS Hybrid Nodes vs EKS Anywhere](03-comparison/README.md)
  * [철학적 차이](03-comparison/philosophy.md)
  * [핵심 차이점 비교](03-comparison/key-differences.md)
  * [선택 가이드](03-comparison/decision-guide.md)
* [EKS Hybrid Nodes](04-hybrid-nodes/README.md)
  * [작동 원리](04-hybrid-nodes/how-it-works.md)
  * [장점과 활용 사례](04-hybrid-nodes/benefits.md)
  * [주의사항과 제약](04-hybrid-nodes/considerations.md)
* [EKS Anywhere](05-anywhere/README.md)
  * [작동 원리](05-anywhere/how-it-works.md)
  * [장점과 활용 사례](05-anywhere/benefits.md)
  * [주의사항과 제약](05-anywhere/considerations.md)

## Part 3: 실전 운영

* [Multi-Cluster 운영](06-multi-cluster/README.md)
  * [왜 여러 클러스터가 필요한가](06-multi-cluster/why-multi-cluster.md)
  * [관리 도구 비교](06-multi-cluster/management-tools.md)
  * [GitOps 전략](06-multi-cluster/gitops.md)
* [보안](07-security/README.md)
  * [신원 및 접근 관리](07-security/identity-access.md)
  * [네트워크 보안](07-security/network-security.md)
  * [워크로드 보안](07-security/workload-security.md)
  * [데이터 보안](07-security/data-security.md)
* [네트워크](08-networking/README.md)
  * [Hybrid Nodes 네트워크 구성](08-networking/hybrid-nodes-network.md)
  * [연결 방식: VPN vs Direct Connect](08-networking/connectivity.md)
  * [CNI 선택: Cilium vs Calico](08-networking/cni-selection.md)
  * [방화벽 규칙 설정](08-networking/firewall-rules.md)
* [운영 관리](09-operations/README.md)
  * [Day-2 운영이란](09-operations/day2-operations.md)
  * [업그레이드 전략](09-operations/upgrades.md)
  * [모니터링 스택 구성](09-operations/monitoring.md)
  * [백업 및 재해복구](09-operations/backup-dr.md)
* [비용 분석](10-cost/README.md)
  * [EKS Hybrid Nodes 비용](10-cost/hybrid-nodes-pricing.md)
  * [EKS Anywhere 비용](10-cost/anywhere-pricing.md)
  * [TCO 분석 및 비교](10-cost/tco-analysis.md)

## Part 4: 트렌드와 시작하기

* [2024-2025 트렌드](11-trends/README.md)
  * [Platform Engineering](11-trends/platform-engineering.md)
  * [GenAI 워크로드](11-trends/genai-workloads.md)
  * [eBPF와 Service Mesh](11-trends/ebpf-service-mesh.md)
* [시작하기](12-getting-started/README.md)
  * [EKS Hybrid Nodes 체크리스트](12-getting-started/hybrid-nodes-checklist.md)
  * [EKS Anywhere 체크리스트](12-getting-started/anywhere-checklist.md)
  * [공통 Best Practices](12-getting-started/best-practices.md)
* [참고 자료](13-references/README.md)
  * [자주 묻는 질문 (FAQ)](13-references/faq.md)
