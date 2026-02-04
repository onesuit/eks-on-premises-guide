# 신원 및 접근 관리

> "누가 클러스터에 접근할 수 있고, 무엇을 할 수 있나요?"

접근 관리는 보안의 기본입니다. Kubernetes에서는 RBAC을 통해 세밀한 권한 제어가 가능합니다.

## RBAC 기본 개념

```
┌─────────────────────────────────────────────────────────────────┐
│                         RBAC 구성요소                           │
│                                                                 │
│   Subject (누가)          Verb (무엇을)      Resource (대상)   │
│   ┌─────────────┐        ┌─────────────┐    ┌─────────────┐    │
│   │ User        │        │ get         │    │ pods        │    │
│   │ Group       │───────►│ list        │───►│ services    │    │
│   │ ServiceAcct │        │ create      │    │ secrets     │    │
│   └─────────────┘        │ delete      │    │ deployments │    │
│                          │ watch       │    │ ...         │    │
│                          └─────────────┘    └─────────────┘    │
│                                                                 │
│   Role / ClusterRole: 권한 정의                                │
│   RoleBinding / ClusterRoleBinding: Subject에 권한 부여        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## Role과 RoleBinding

### Namespace 범위 권한 (Role)

```yaml
# 특정 네임스페이스에서 Pod 읽기 권한
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: production
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get"]
---
# 역할을 사용자에게 부여
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

### 클러스터 범위 권한 (ClusterRole)

```yaml
# 모든 네임스페이스에서 노드 정보 읽기
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-nodes
subjects:
- kind: Group
  name: sre-team
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: node-reader
  apiGroup: rbac.authorization.k8s.io
```

## 역할 기반 권한 설계

```
┌─────────────────────────────────────────────────────────────────┐
│                    역할별 권한 예시                             │
│                                                                 │
│   Cluster Admin                                                 │
│   └── 모든 리소스에 대한 전체 권한                             │
│                                                                 │
│   Namespace Admin                                               │
│   └── 특정 네임스페이스 전체 권한                              │
│                                                                 │
│   Developer                                                     │
│   ├── Deployment, Pod, Service, ConfigMap: 전체               │
│   ├── Secret: 읽기만                                           │
│   └── Node, PV: 읽기만                                         │
│                                                                 │
│   Viewer                                                        │
│   └── 모든 리소스 읽기만                                       │
│                                                                 │
│   CI/CD Service Account                                         │
│   ├── Deployment: create, update                               │
│   ├── Secret: 읽기 (배포용)                                    │
│   └── 기타: 제한적                                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 역할 정의 예시

```yaml
# Developer 역할
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: development
  name: developer
rules:
- apiGroups: ["", "apps", "batch"]
  resources: ["deployments", "pods", "services", "configmaps", "jobs"]
  verbs: ["*"]
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list"]  # 읽기만
- apiGroups: [""]
  resources: ["pods/exec", "pods/portforward"]
  verbs: ["create"]  # 디버깅용
```

## EKS Hybrid Nodes: IAM 통합

EKS Hybrid Nodes는 AWS IAM과 Kubernetes RBAC을 연동할 수 있습니다.

```yaml
# aws-auth ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    - rolearn: arn:aws:iam::123456789:role/DeveloperRole
      username: developer
      groups:
        - developers
    - rolearn: arn:aws:iam::123456789:role/AdminRole
      username: admin
      groups:
        - system:masters
```

### Pod Identity (Hybrid Nodes)

```yaml
# ServiceAccount에 IAM Role 연결
apiVersion: v1
kind: ServiceAccount
metadata:
  name: s3-reader
  namespace: default
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/S3ReaderRole
---
# Pod에서 사용
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  serviceAccountName: s3-reader
  containers:
  - name: app
    image: my-app
    # AWS SDK가 자동으로 IAM 역할 사용
```

## EKS Anywhere: OIDC 설정

EKS Anywhere에서는 OIDC를 직접 구성해야 합니다.

```yaml
# cluster-config.yaml OIDC 설정
apiVersion: anywhere.eks.amazonaws.com/v1alpha1
kind: Cluster
metadata:
  name: my-cluster
spec:
  identityProviderRefs:
    - kind: OIDCConfig
      name: my-oidc
---
apiVersion: anywhere.eks.amazonaws.com/v1alpha1
kind: OIDCConfig
metadata:
  name: my-oidc
spec:
  clientId: kubernetes
  groupsClaim: groups
  groupsPrefix: "oidc:"
  issuerUrl: https://auth.example.com
  usernameClaim: email
  usernamePrefix: "oidc:"
```

## 모범 사례

```
┌─────────────────────────────────────────────────────────────────┐
│                    RBAC 모범 사례                               │
│                                                                 │
│   최소 권한 원칙:                                               │
│   ✅ 필요한 권한만 부여                                        │
│   ✅ 와일드카드(*) 사용 최소화                                 │
│   ✅ cluster-admin은 극소수에게만                              │
│                                                                 │
│   그룹 활용:                                                    │
│   ✅ 개인이 아닌 그룹에 권한 부여                              │
│   ✅ 그룹 멤버십으로 관리                                      │
│                                                                 │
│   네임스페이스 분리:                                            │
│   ✅ 팀/환경별 네임스페이스                                    │
│   ✅ Role은 네임스페이스 범위 선호                             │
│                                                                 │
│   정기 감사:                                                    │
│   ✅ 권한 정기 검토                                            │
│   ✅ 미사용 계정/역할 삭제                                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

[다음: 네트워크 보안 →](network-security.md)
