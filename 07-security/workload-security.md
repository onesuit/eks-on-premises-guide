# 워크로드 보안

> "컨테이너가 root로 실행되면 안 되는 거죠?"

컨테이너와 Pod 수준의 보안은 매우 중요합니다. 잘못 구성된 Pod 하나가 전체 클러스터를 위험에 빠뜨릴 수 있습니다.

## Pod Security Standards (PSS)

Kubernetes 1.25+에서는 Pod Security Standards가 기본입니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Pod Security Standards 수준                  │
│                                                                 │
│   Privileged (가장 느슨)                                       │
│   └── 모든 것 허용, 제한 없음                                  │
│                                                                 │
│   Baseline (기본 권장)                                         │
│   ├── hostNetwork, hostPID 차단                                │
│   ├── 특권 컨테이너 차단                                       │
│   └── 알려진 위험 설정 차단                                    │
│                                                                 │
│   Restricted (가장 엄격)                                       │
│   ├── Baseline 포함                                            │
│   ├── root 실행 차단                                           │
│   ├── 특정 볼륨 타입만 허용                                    │
│   └── seccomp 필수                                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### PSS 적용

```yaml
# 네임스페이스에 PSS 적용
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

### 안전한 Pod 예시

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: my-app:latest
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
          - ALL
    resources:
      limits:
        memory: "128Mi"
        cpu: "500m"
      requests:
        memory: "64Mi"
        cpu: "250m"
```

## 이미지 보안

### 취약점 스캐닝

```yaml
# Trivy로 이미지 스캐닝 (CI/CD 예시)
# .github/workflows/scan.yaml
name: Image Scan
on: push
jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
    - uses: aquasecurity/trivy-action@master
      with:
        image-ref: 'my-app:${{ github.sha }}'
        format: 'table'
        exit-code: '1'
        severity: 'CRITICAL,HIGH'
```

### 이미지 서명 (Sigstore/Cosign)

```bash
# 이미지 서명
cosign sign --key cosign.key my-registry/my-app:v1.0.0

# 서명 검증
cosign verify --key cosign.pub my-registry/my-app:v1.0.0
```

## OPA/Gatekeeper

더 세밀한 정책 제어를 위해 OPA Gatekeeper를 사용할 수 있습니다.

```yaml
# 허용된 레지스트리에서만 이미지 Pull
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: allowedrepos
spec:
  crd:
    spec:
      names:
        kind: AllowedRepos
      validation:
        openAPIV3Schema:
          type: object
          properties:
            repos:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package allowedrepos
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          satisfied := [good | repo = input.parameters.repos[_]; good = startswith(container.image, repo)]
          not any(satisfied)
          msg := sprintf("container <%v> has an invalid image repo <%v>", [container.name, container.image])
        }
---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: AllowedRepos
metadata:
  name: allowed-repos
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
    namespaces: ["production"]
  parameters:
    repos:
      - "123456789.dkr.ecr.ap-northeast-2.amazonaws.com/"
      - "my-harbor.example.com/"
```

## 런타임 보안

### Falco

```yaml
# Falco 규칙 예시: 컨테이너에서 쉘 실행 탐지
- rule: Terminal shell in container
  desc: A shell was spawned in a container
  condition: >
    spawned_process and 
    container and
    shell_procs and
    proc.pname exists and
    not proc.pname in (shell_binaries)
  output: >
    Shell spawned in container 
    (user=%user.name container=%container.name shell=%proc.name)
  priority: WARNING
```

## 모범 사례 체크리스트

```
┌─────────────────────────────────────────────────────────────────┐
│                 워크로드 보안 체크리스트                        │
│                                                                 │
│   Pod 설정:                                                     │
│   □ runAsNonRoot: true                                         │
│   □ readOnlyRootFilesystem: true                               │
│   □ allowPrivilegeEscalation: false                            │
│   □ capabilities 모두 drop                                     │
│   □ resource limits 설정                                        │
│                                                                 │
│   이미지:                                                       │
│   □ 최신 베이스 이미지 사용                                    │
│   □ 취약점 스캐닝 통과                                         │
│   □ 신뢰할 수 있는 레지스트리만 사용                           │
│   □ 이미지 태그 대신 digest 사용                               │
│                                                                 │
│   네임스페이스:                                                 │
│   □ PSS restricted 또는 baseline 적용                          │
│   □ ResourceQuota 설정                                          │
│   □ LimitRange 설정                                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

[다음: 데이터 보안 →](data-security.md)
