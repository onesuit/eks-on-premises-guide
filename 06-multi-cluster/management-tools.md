# 관리 도구 비교

> "여러 클러스터를 어떻게 효율적으로 관리하나요?"

Multi-Cluster 환경에서는 적절한 도구 없이는 운영이 불가능합니다. 주요 도구들을 비교해봅시다.

## 도구 카테고리

```
┌─────────────────────────────────────────────────────────────────┐
│                    Multi-Cluster 관리 도구                      │
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  GitOps 도구                                             │  │
│   │  ├── ArgoCD                                             │  │
│   │  └── Flux                                               │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  클러스터 관리 플랫폼                                   │  │
│   │  ├── Rancher                                            │  │
│   │  └── Red Hat ACM                                        │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  AWS 네이티브                                           │  │
│   │  └── EKS Connector                                      │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## ArgoCD

### 개요

ArgoCD는 가장 인기 있는 GitOps 도구 중 하나입니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                        ArgoCD Architecture                       │
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │                    ArgoCD Server                         │  │
│   │                                                          │  │
│   │   ┌──────────────────────────────────────────────────┐  │  │
│   │   │  • Application Controller                        │  │  │
│   │   │  • Repo Server                                   │  │  │
│   │   │  • API Server                                    │  │  │
│   │   │  • Web UI                                        │  │  │
│   │   └──────────────────────────────────────────────────┘  │  │
│   │                          │                               │  │
│   └──────────────────────────┼───────────────────────────────┘  │
│                              │                                  │
│              ┌───────────────┼───────────────┐                 │
│              │               │               │                 │
│              ▼               ▼               ▼                 │
│       ┌──────────┐    ┌──────────┐    ┌──────────┐            │
│       │ Cluster1 │    │ Cluster2 │    │ Cluster3 │            │
│       └──────────┘    └──────────┘    └──────────┘            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 핵심 기능

```yaml
# ApplicationSet 예시 - 여러 클러스터에 동시 배포
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: my-app
  namespace: argocd
spec:
  generators:
  - clusters:
      selector:
        matchLabels:
          env: production
  template:
    metadata:
      name: '{{name}}-my-app'
    spec:
      project: default
      source:
        repoURL: https://github.com/org/repo
        targetRevision: HEAD
        path: apps/my-app
      destination:
        server: '{{server}}'
        namespace: my-app
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

### 장단점

| 장점 | 단점 |
|------|------|
| 직관적인 Web UI | 복잡한 초기 설정 |
| 강력한 ApplicationSet | 메모리 사용량 높음 |
| 활발한 커뮤니티 | SSO 설정 복잡 |
| RBAC 지원 | Helm 통합 제한적 |

## Flux

### 개요

Flux는 CNCF 프로젝트이며, EKS Anywhere에 기본 내장되어 있습니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                        Flux Architecture                         │
│                                                                 │
│   각 클러스터에서 독립 실행                                    │
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  Cluster 1                                               │  │
│   │  ┌───────────────────────────────────────────────────┐  │  │
│   │  │  Flux Controllers                                  │  │  │
│   │  │  ├── source-controller                            │  │  │
│   │  │  ├── kustomize-controller                         │  │  │
│   │  │  └── helm-controller                              │  │  │
│   │  └───────────────────────────────────────────────────┘  │  │
│   └─────────────────────────────────────────────────────────┘  │
│                              ▲                                  │
│                              │                                  │
│                         Git Repository                          │
│                              │                                  │
│                              ▼                                  │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  Cluster 2 (동일 구조)                                   │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 핵심 기능

```yaml
# Flux Kustomization 예시
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: my-app
  namespace: flux-system
spec:
  interval: 5m
  path: ./apps/my-app
  prune: true
  sourceRef:
    kind: GitRepository
    name: my-repo
  targetNamespace: my-app
  healthChecks:
  - apiVersion: apps/v1
    kind: Deployment
    name: my-app
    namespace: my-app
---
# HelmRelease 예시
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: nginx
spec:
  interval: 5m
  chart:
    spec:
      chart: nginx
      sourceRef:
        kind: HelmRepository
        name: bitnami
  values:
    replicaCount: 3
```

### 장단점

| 장점 | 단점 |
|------|------|
| EKS Anywhere 기본 내장 | Web UI 없음 |
| 가벼움, 리소스 적게 사용 | 시각화 어려움 |
| CLI 중심 | 학습 곡선 |
| 우수한 Helm 통합 | 디버깅 어려움 |

## 도구 비교표

| 기능 | ArgoCD | Flux |
|------|--------|------|
| **Web UI** | ✅ 풍부함 | ❌ 없음 |
| **CLI** | ✅ | ✅ |
| **Multi-tenancy** | ✅ Projects | ✅ Namespaces |
| **Helm 지원** | ✅ | ✅ (더 나음) |
| **Kustomize** | ✅ | ✅ |
| **RBAC** | ✅ | Kubernetes RBAC |
| **SSO** | ✅ (복잡) | Kubernetes OIDC |
| **Notifications** | ✅ | ✅ |
| **Image Automation** | ❌ (Argo Image) | ✅ |
| **리소스 사용량** | 높음 | 낮음 |

## 선택 가이드

### ArgoCD를 선택하세요

```
✅ Web UI가 중요하다
✅ 중앙 집중식 관리 선호
✅ 여러 팀이 사용 (시각화 필요)
✅ 복잡한 배포 파이프라인
✅ 승인 프로세스 필요
```

### Flux를 선택하세요

```
✅ EKS Anywhere 사용 (기본 내장)
✅ CLI 중심 워크플로우
✅ 리소스 제약 있음
✅ Helm 중심 배포
✅ 각 클러스터 독립 관리
```

## 하이브리드 접근

둘 다 사용하는 것도 가능합니다:

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   하이브리드 구성 예시                                         │
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │               ArgoCD (중앙 관리 클러스터)               │  │
│   │                                                          │  │
│   │   역할:                                                  │  │
│   │   • 전체 클러스터 가시성                                │  │
│   │   • 대시보드 및 UI                                      │  │
│   │   • 승인 워크플로우                                     │  │
│   │                                                          │  │
│   └─────────────────────────────────────────────────────────┘  │
│                              │                                  │
│          ┌───────────────────┼───────────────────┐             │
│          ▼                   ▼                   ▼             │
│   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐        │
│   │ EKS Anywhere │  │ EKS Anywhere │  │ EKS Anywhere │        │
│   │ + Flux       │  │ + Flux       │  │ + Flux       │        │
│   │              │  │              │  │              │        │
│   │ 역할:       │  │ 역할:       │  │ 역할:       │        │
│   │ • 실제 배포 │  │ • 실제 배포 │  │ • 실제 배포 │        │
│   │ • 로컬 동기화│  │ • 로컬 동기화│  │ • 로컬 동기화│        │
│   └──────────────┘  └──────────────┘  └──────────────┘        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

