# GitOps 전략

> "Git에 Push하면 자동으로 배포된다고요?"

GitOps는 Git을 "Single Source of Truth"로 사용하여 인프라와 애플리케이션을 관리하는 방식입니다. Multi-Cluster 환경에서 특히 빛을 발합니다.

## GitOps 핵심 원칙

```
┌─────────────────────────────────────────────────────────────────┐
│                    GitOps 4가지 원칙                            │
│                                                                 │
│   1. 선언적 (Declarative)                                      │
│      └── 원하는 상태를 YAML로 선언                             │
│                                                                 │
│   2. 버전 관리 (Versioned)                                     │
│      └── 모든 변경은 Git에 기록                                │
│                                                                 │
│   3. 자동 적용 (Automatically Applied)                         │
│      └── Git 변경 감지 → 자동 배포                             │
│                                                                 │
│   4. 지속적 조정 (Continuously Reconciled)                     │
│      └── 실제 상태가 Git과 다르면 자동 복구                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## Git 저장소 구조

### 모노 레포 전략

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   gitops-repo/                                                  │
│   ├── clusters/                    # 클러스터별 설정           │
│   │   ├── dev/                                                 │
│   │   │   ├── flux-system/                                     │
│   │   │   └── kustomization.yaml                               │
│   │   ├── staging/                                             │
│   │   │   ├── flux-system/                                     │
│   │   │   └── kustomization.yaml                               │
│   │   └── prod/                                                │
│   │       ├── flux-system/                                     │
│   │       └── kustomization.yaml                               │
│   │                                                             │
│   ├── apps/                        # 애플리케이션              │
│   │   ├── base/                    # 공통 설정                 │
│   │   │   ├── deployment.yaml                                  │
│   │   │   ├── service.yaml                                     │
│   │   │   └── kustomization.yaml                               │
│   │   └── overlays/                # 환경별 오버레이          │
│   │       ├── dev/                                             │
│   │       ├── staging/                                         │
│   │       └── prod/                                            │
│   │                                                             │
│   └── infrastructure/              # 인프라 컴포넌트           │
│       ├── cert-manager/                                        │
│       ├── ingress-nginx/                                       │
│       └── monitoring/                                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 멀티 레포 전략

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   Repo 1: platform-config         Repo 2: app-frontend         │
│   ┌─────────────────────┐        ┌─────────────────────┐       │
│   │ 클러스터 설정       │        │ Frontend 앱         │       │
│   │ 인프라 컴포넌트     │        │ 배포 설정           │       │
│   │ 공통 정책           │        │                     │       │
│   └─────────────────────┘        └─────────────────────┘       │
│                                                                 │
│   Repo 3: app-backend             Repo 4: app-ml                │
│   ┌─────────────────────┐        ┌─────────────────────┐       │
│   │ Backend 앱          │        │ ML 워크로드         │       │
│   │ 배포 설정           │        │ 배포 설정           │       │
│   │                     │        │                     │       │
│   └─────────────────────┘        └─────────────────────┘       │
│                                                                 │
│   장점: 팀별 독립성, 세분화된 접근 제어                        │
│   단점: 관리 복잡도 증가                                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## 실전 예제

### 환경별 배포 (Kustomize)

```yaml
# apps/base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: app
        image: my-app:latest
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
---
# apps/base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
```

```yaml
# apps/overlays/prod/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
patchesStrategicMerge:
  - deployment-patch.yaml
---
# apps/overlays/prod/deployment-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 5                    # Prod는 5개
  template:
    spec:
      containers:
      - name: app
        resources:
          requests:
            memory: "256Mi"      # 더 많은 리소스
            cpu: "500m"
          limits:
            memory: "512Mi"
            cpu: "1000m"
```

### Progressive Delivery

```yaml
# Flagger를 사용한 카나리 배포
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: my-app
  namespace: prod
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  service:
    port: 80
  analysis:
    interval: 30s
    threshold: 5
    maxWeight: 50
    stepWeight: 10
    metrics:
    - name: request-success-rate
      thresholdRange:
        min: 99
    - name: request-duration
      thresholdRange:
        max: 500
```

```
배포 진행:
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   Step 1:  Stable v1: 100%  │  Canary v2: 0%                   │
│                                                                 │
│   Step 2:  Stable v1: 90%   │  Canary v2: 10%  ← 메트릭 체크   │
│                                                                 │
│   Step 3:  Stable v1: 80%   │  Canary v2: 20%  ← 메트릭 OK     │
│                                                                 │
│   ...                                                           │
│                                                                 │
│   Step N:  Stable v2: 100%  │  Canary: 삭제   ← 완료           │
│                                                                 │
│   또는 롤백:                                                    │
│   Step X:  메트릭 실패 → 자동 롤백 → Stable v1: 100%          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## 승인 워크플로우

### Pull Request 기반 배포

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   개발자                                                        │
│      │                                                          │
│      │  1. 변경 사항 커밋                                      │
│      ▼                                                          │
│   Feature Branch ──────► Pull Request                          │
│                              │                                  │
│                              │  2. 자동 검증                    │
│                              │  ├── YAML 문법 검사             │
│                              │  ├── 정책 검사 (OPA)            │
│                              │  └── Diff 미리보기              │
│                              │                                  │
│                              │  3. 수동 승인                    │
│                              │  └── 팀 리드 approve            │
│                              │                                  │
│                              ▼                                  │
│                          Merge to Main                          │
│                              │                                  │
│                              │  4. 자동 배포                    │
│                              ▼                                  │
│                    ArgoCD/Flux 감지 → 클러스터 적용            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 환경별 승인 정책

```yaml
# GitHub CODEOWNERS 예시
# 환경별 다른 승인자 필요

/clusters/dev/*       @dev-team
/clusters/staging/*   @qa-team @dev-leads
/clusters/prod/*      @sre-team @security-team
```

## 시크릿 관리

GitOps에서 시크릿은 특별한 처리가 필요합니다.

### 방법 1: Sealed Secrets

```yaml
# 암호화된 시크릿을 Git에 저장
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: my-secret
spec:
  encryptedData:
    password: AgBy8hCi...암호화된 데이터...
```

### 방법 2: External Secrets Operator

```yaml
# 외부 시크릿 관리자 참조
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: my-secret
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: SecretStore
  target:
    name: my-secret
  data:
  - secretKey: password
    remoteRef:
      key: prod/my-app/password
```

### 방법 3: SOPS (Mozilla)

```yaml
# sops로 암호화된 YAML
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
stringData:
  password: ENC[AES256_GCM,data:...,type:str]
sops:
  kms:
    - arn: arn:aws:kms:...
```

## 모범 사례

```
┌─────────────────────────────────────────────────────────────────┐
│                    GitOps 모범 사례                             │
│                                                                 │
│   Do:                                                           │
│   ✅ 모든 변경은 Git을 통해                                    │
│   ✅ 환경별 브랜치 또는 디렉토리 분리                          │
│   ✅ PR 리뷰 프로세스 적용                                     │
│   ✅ 시크릿은 암호화 또는 외부 참조                            │
│   ✅ 변경 전 Dry-run 검증                                      │
│                                                                 │
│   Don't:                                                        │
│   ❌ kubectl apply 직접 실행                                   │
│   ❌ 시크릿을 평문으로 Git에 저장                              │
│   ❌ 수동 승인 없이 Prod 배포                                  │
│   ❌ 롤백 계획 없이 배포                                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

