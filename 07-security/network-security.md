# 네트워크 보안

> "Pod끼리 마음대로 통신하면 안 되는 거 아닌가요?"

기본적으로 Kubernetes의 모든 Pod는 서로 통신할 수 있습니다. Network Policy를 통해 이를 제어할 수 있습니다.

## Network Policy 기본

```yaml
# 모든 인바운드 트래픽 차단 (기본 거부)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}  # 모든 Pod에 적용
  policyTypes:
  - Ingress
---
# 특정 Pod에서만 접근 허용
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
```

## 일반적인 Network Policy 패턴

### 패턴 1: 네임스페이스 격리

```yaml
# 같은 네임스페이스 내에서만 통신 허용
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: namespace-isolation
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: production
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: production
  - to:  # DNS 허용
    - namespaceSelector: {}
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
```

### 패턴 2: 3-Tier 아키텍처

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   Internet                                                      │
│       │                                                         │
│       ▼                                                         │
│   ┌─────────┐                                                   │
│   │ Ingress │                                                   │
│   └────┬────┘                                                   │
│        │ (허용)                                                  │
│        ▼                                                         │
│   ┌─────────────────┐                                          │
│   │    Frontend     │ ─── 외부에서 접근 가능                   │
│   │    (Pods)       │                                          │
│   └────────┬────────┘                                          │
│            │ (허용)                                              │
│            ▼                                                     │
│   ┌─────────────────┐                                          │
│   │    Backend      │ ─── Frontend에서만 접근 가능             │
│   │    (Pods)       │                                          │
│   └────────┬────────┘                                          │
│            │ (허용)                                              │
│            ▼                                                     │
│   ┌─────────────────┐                                          │
│   │    Database     │ ─── Backend에서만 접근 가능              │
│   │    (Pods)       │                                          │
│   └─────────────────┘                                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```yaml
# Database는 Backend에서만 접근
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-policy
spec:
  podSelector:
    matchLabels:
      tier: database
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: backend
    ports:
    - protocol: TCP
      port: 5432
```

## Cilium Network Policy

EKS Anywhere와 Hybrid Nodes는 Cilium을 사용할 수 있으며, 더 강력한 기능을 제공합니다.

```yaml
# Cilium: Layer 7 정책 (HTTP 메서드 기반)
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: l7-rule
spec:
  endpointSelector:
    matchLabels:
      app: api
  ingress:
  - fromEndpoints:
    - matchLabels:
        app: frontend
    toPorts:
    - ports:
      - port: "80"
        protocol: TCP
      rules:
        http:
        - method: "GET"
          path: "/api/public/.*"
        - method: "POST"
          path: "/api/public/.*"
          headers:
          - 'X-Auth-Token: [a-zA-Z0-9]+'
```

## Service Mesh (선택)

### Istio mTLS

```yaml
# 네임스페이스 전체에 mTLS 적용
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT
```

### Istio Authorization

```yaml
# 서비스 간 접근 제어
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: backend-policy
  namespace: production
spec:
  selector:
    matchLabels:
      app: backend
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/production/sa/frontend"]
    to:
    - operation:
        methods: ["GET", "POST"]
```

## Egress 제어

```yaml
# 외부 통신 제한
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: limit-egress
spec:
  podSelector:
    matchLabels:
      app: secure-app
  policyTypes:
  - Egress
  egress:
  - to:
    - ipBlock:
        cidr: 10.0.0.0/8      # 내부 네트워크만
  - to:                        # 특정 외부 API만
    - ipBlock:
        cidr: 54.239.28.85/32  # api.example.com
    ports:
    - protocol: TCP
      port: 443
```

## 모범 사례

```
┌─────────────────────────────────────────────────────────────────┐
│                 네트워크 보안 모범 사례                         │
│                                                                 │
│   기본 거부:                                                    │
│   ✅ 모든 네임스페이스에 default-deny 정책                     │
│   ✅ 필요한 트래픽만 명시적 허용                               │
│                                                                 │
│   최소 권한:                                                    │
│   ✅ 필요한 포트만 허용                                        │
│   ✅ 필요한 Pod에서만 접근 허용                                │
│                                                                 │
│   계층별 분리:                                                  │
│   ✅ Frontend, Backend, Database 분리                          │
│   ✅ 네임스페이스별 격리                                       │
│                                                                 │
│   모니터링:                                                     │
│   ✅ 네트워크 플로우 로깅                                      │
│   ✅ 이상 트래픽 탐지                                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

[다음: 워크로드 보안 →](workload-security.md)
