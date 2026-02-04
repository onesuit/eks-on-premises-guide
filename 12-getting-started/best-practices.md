# 공통 Best Practices

> EKS Hybrid Nodes와 EKS Anywhere 모두에 적용되는 운영 Best Practices입니다.

---

## 아키텍처 설계

### 고가용성 (HA) 구성

```
프로덕션 환경에서는 반드시 고가용성을 고려하세요:

┌─────────────────────────────────────────────────────────────────┐
│                    고가용성 아키텍처                             │
│                                                                 │
│  EKS Hybrid Nodes:                                              │
│  ├── Control Plane: AWS가 자동으로 HA 구성 (걱정 불필요)       │
│  └── Worker Nodes: 최소 3대 이상, 여러 랙/존에 분산            │
│                                                                 │
│  EKS Anywhere:                                                  │
│  ├── Control Plane: 반드시 3대 (etcd quorum을 위해)            │
│  ├── Worker Nodes: 최소 3대 이상                               │
│  └── 물리적 분산: 다른 랙, 다른 전원, 다른 네트워크 스위치    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**왜 3대인가?**

```
etcd quorum 공식: (n/2) + 1

노드 수 | quorum | 허용 장애 노드 | 권장 여부
--------|--------|---------------|----------
1       | 1      | 0             | ❌ 개발만
2       | 2      | 0             | ❌ 의미없음
3       | 2      | 1             | ✓ 최소 프로덕션
5       | 3      | 2             | ✓ 대규모 환경
7       | 4      | 3             | 드물게 필요

※ 짝수 노드(4, 6)는 장애 허용 수가 홀수와 같아서 비효율적
```

### 노드 분리 전략

```yaml
# 워크로드 특성에 따른 노드 분리

# 1. 노드에 라벨 부여
$ kubectl label node worker-1 workload-type=general
$ kubectl label node worker-2 workload-type=general
$ kubectl label node worker-3 workload-type=memory-intensive
$ kubectl label node worker-4 workload-type=gpu

# 2. Pod에서 nodeSelector 사용
apiVersion: v1
kind: Pod
spec:
  nodeSelector:
    workload-type: memory-intensive  # 메모리 집약 워크로드
  containers:
  - name: app
    image: my-app
    resources:
      requests:
        memory: "32Gi"
```

---

## 보안

### 최소 권한 원칙 적용

```yaml
# 1. 네임스페이스별 RBAC
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: team-a
  name: developer
rules:
- apiGroups: ["", "apps"]
  resources: ["pods", "deployments", "services"]
  verbs: ["get", "list", "watch", "create", "update", "delete"]
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list"]  # Secret 생성/수정은 제한
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: team-a-developers
  namespace: team-a
subjects:
- kind: Group
  name: team-a-developers
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
```

### Network Policy 기본 적용

```yaml
# 기본: 모든 트래픽 차단 (Deny All)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}  # 모든 Pod에 적용
  policyTypes:
  - Ingress
  - Egress
---
# 필요한 트래픽만 명시적으로 허용
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
```

### Pod Security Standards 적용

```yaml
# 네임스페이스에 보안 정책 적용
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/audit: restricted
```

**Restricted 정책 요구사항:**

```yaml
# restricted 정책을 만족하는 Pod 예시
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    runAsNonRoot: true
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: my-app
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
    resources:
      limits:
        cpu: "500m"
        memory: "128Mi"
      requests:
        cpu: "100m"
        memory: "64Mi"
```

---

## 리소스 관리

### Resource Requests/Limits 필수 설정

```yaml
# 모든 컨테이너에 리소스 설정 권장
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: app
    resources:
      requests:
        cpu: "100m"      # 최소 보장
        memory: "128Mi"
      limits:
        cpu: "500m"      # 최대 사용 가능
        memory: "512Mi"
```

**가이드라인:**

```
리소스 설정 전략:

requests:
├── 평상시 사용량 기준으로 설정
├── 스케줄러가 노드 배치에 사용
└── 너무 낮으면 노드에 Pod가 과밀 배치

limits:
├── 최대 허용량 설정
├── CPU: 초과 시 throttling (죽지 않음)
├── Memory: 초과 시 OOMKilled (죽음!)
└── 너무 높으면 의미 없음

권장 비율:
├── CPU: limits = requests × 2~5
└── Memory: limits = requests × 1.5~2
```

### LimitRange로 기본값 강제

```yaml
# 네임스페이스에 기본 리소스 제한 설정
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: production
spec:
  limits:
  - default:           # 기본 limits
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:    # 기본 requests
      cpu: "100m"
      memory: "128Mi"
    max:               # 최대 허용
      cpu: "2"
      memory: "4Gi"
    min:               # 최소 요구
      cpu: "50m"
      memory: "64Mi"
    type: Container
```

### ResourceQuota로 네임스페이스 할당량 관리

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-a-quota
  namespace: team-a
spec:
  hard:
    requests.cpu: "10"
    requests.memory: "20Gi"
    limits.cpu: "20"
    limits.memory: "40Gi"
    pods: "50"
    services: "10"
    persistentvolumeclaims: "10"
```

---

## 모니터링 & 알람

### 필수 모니터링 메트릭

```
클러스터 레벨:
├── 노드 수 및 상태
├── API Server 응답 시간
├── etcd 상태 (EKS Anywhere)
└── 전체 리소스 사용률

노드 레벨:
├── CPU/Memory 사용률
├── 디스크 사용률 (특히 /var/lib/containerd)
├── 네트워크 I/O
└── kubelet 상태

Pod 레벨:
├── 재시작 횟수
├── 리소스 사용률 vs 요청량
├── 응답 시간 (애플리케이션)
└── 에러율
```

### 필수 알람 설정

```yaml
# Prometheus AlertManager 규칙 예시
groups:
- name: kubernetes-alerts
  rules:
  # 노드 다운
  - alert: NodeNotReady
    expr: kube_node_status_condition{condition="Ready",status="true"} == 0
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "Node {{ $labels.node }} is not ready"
      
  # 높은 메모리 사용률
  - alert: HighMemoryUsage
    expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) > 0.9
    for: 10m
    labels:
      severity: warning
    annotations:
      summary: "High memory usage on {{ $labels.instance }}"
      
  # Pod 재시작 반복
  - alert: PodCrashLooping
    expr: rate(kube_pod_container_status_restarts_total[15m]) > 0
    for: 15m
    labels:
      severity: warning
    annotations:
      summary: "Pod {{ $labels.pod }} is crash looping"
      
  # 디스크 공간 부족
  - alert: DiskSpaceLow
    expr: (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) < 0.1
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "Low disk space on {{ $labels.instance }}"
```

### 로그 관리

```
로그 보관 전략:

애플리케이션 로그:
├── 보관 기간: 30일 (조정 가능)
├── 수집: Fluent Bit / Promtail
└── 저장: CloudWatch Logs / Loki / Elasticsearch

감사 로그 (Audit Log):
├── 보관 기간: 90일~1년 (규제에 따라)
├── 변경 불가능한 저장소 권장
└── 정기 검토 프로세스 수립

시스템 로그:
├── kubelet, containerd 로그
├── 보관 기간: 7일
└── 디스크 공간 관리 주의
```

---

## 백업 & 복구

### 백업 대상 및 주기

```
┌─────────────────────────────────────────────────────────────────┐
│                    백업 전략                                     │
│                                                                 │
│  대상              │ 주기        │ 보관        │ 방법          │
│  ─────────────────────────────────────────────────────────────│
│  etcd (EKS Anywhere)│ 6시간      │ 7일         │ etcdctl      │
│  Kubernetes 리소스  │ 매일       │ 30일        │ Velero       │
│  PV 데이터         │ 매일       │ 30일        │ Velero + CSI │
│  설정 파일/IaC     │ 변경 시    │ 영구        │ Git          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 복구 테스트

```
정기 복구 훈련 (분기별 권장):

□ 시나리오 1: 단일 Pod 복구
  └── Velero restore로 특정 Deployment 복구

□ 시나리오 2: 네임스페이스 복구
  └── 전체 네임스페이스 삭제 후 Velero 복구

□ 시나리오 3: 클러스터 재해 복구 (EKS Anywhere)
  └── etcd 백업에서 새 클러스터로 복구

□ 결과 문서화
  └── 복구 시간, 발견된 문제점, 개선 사항
```

---

## 업그레이드 전략

### 업그레이드 계획

```
업그레이드 전:
├── Release Notes 검토
├── 호환성 확인 (애플리케이션, Add-ons)
├── 테스트 환경에서 먼저 검증
├── 백업 완료 확인
└── 롤백 계획 수립

업그레이드 순서:
1. 테스트 환경 업그레이드 → 검증 (1-2주)
2. Staging 환경 업그레이드 → 검증 (1-2주)
3. Production 업그레이드 (유지보수 윈도우)

업그레이드 후:
├── 모든 노드 상태 확인
├── 핵심 애플리케이션 동작 확인
├── 모니터링 메트릭 정상 확인
└── 롤백 대기 (24-48시간)
```

### 버전 지원 정책

```
Kubernetes 버전 지원:

AWS EKS:
├── 표준 지원: 14개월
├── 연장 지원: 추가 12개월 (비용 발생)
└── 권장: N-1 버전 유지 (최신에서 한 단계 이전)

업그레이드 주기:
├── Minor 버전: 연 1-2회
├── Patch 버전: 보안 이슈 시 즉시
└── 최대 2 버전 뒤처지지 않도록 관리
```

---

## 변경 관리

### GitOps 적용

```
모든 변경은 Git을 통해:

┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  Developer ──► Git Repository ──► ArgoCD/Flux ──► Cluster      │
│                     │                                           │
│                     │                                           │
│              ┌──────▼──────┐                                   │
│              │   Review    │  ← PR 리뷰 필수                   │
│              │   Approve   │                                   │
│              │   Merge     │                                   │
│              └─────────────┘                                   │
│                                                                 │
│  장점:                                                          │
│  ├── 변경 이력 추적                                            │
│  ├── 롤백 용이 (git revert)                                   │
│  ├── 코드 리뷰를 통한 품질 관리                               │
│  └── 감사 추적 (누가, 언제, 무엇을)                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 변경 승인 프로세스

```yaml
# 변경 유형별 승인 요구사항

긴급 변경 (P1 장애):
├── 승인자: On-call 엔지니어
├── 사후 리뷰: 필수 (24시간 내)
└── 문서화: 사후 RCA 작성

일반 변경 (기능 배포):
├── 승인자: 팀 리드 1명
├── PR 리뷰: 최소 1명
└── 테스트: CI 파이프라인 통과

중요 변경 (인프라, 보안):
├── 승인자: 팀 리드 + 보안 담당자
├── PR 리뷰: 최소 2명
├── 테스트: Staging 검증 필수
└── 변경 공지: 관련 팀 사전 고지
```

---

## 문서화

### 필수 문서

```
운영 문서:
├── 아키텍처 다이어그램 (최신 상태 유지)
├── 네트워크 토폴로지
├── 접근 권한 매트릭스 (누가 무엇에 접근 가능)
├── 비밀 관리 방안
└── 온보딩 가이드 (신규 팀원용)

절차 문서:
├── 장애 대응 Runbook
├── 업그레이드 절차
├── 백업/복구 절차
├── 보안 사고 대응 절차
└── 온콜 에스컬레이션 절차

변경 기록:
├── 변경 로그 (CHANGELOG)
├── 결정 기록 (ADR - Architecture Decision Records)
└── 장애 보고서 (Postmortem)
```

### 문서 관리

```
문서 위치:
├── Git Repository (코드와 함께)
│   └── /docs 폴더
├── 위키 (Confluence, Notion 등)
│   └── 팀 공유 지식
└── Runbook (PagerDuty, Opsgenie 등)
    └── 장애 대응 자동화

정기 리뷰:
├── 분기별 문서 정확성 검토
├── 변경 발생 시 즉시 업데이트
└── 신규 팀원 온보딩 시 피드백 수집
```

---

## 팀 운영

### 온콜 체계

```
온콜 로테이션:
├── 주간 단위 로테이션 권장
├── 최소 2명 이상 교대
└── 번아웃 방지: 연속 온콜 제한

에스컬레이션:
├── L1: 온콜 엔지니어 (15분 내 응답)
├── L2: 시니어 엔지니어 (30분 내)
├── L3: 팀 리드 / 아키텍트
└── 외부: 벤더 지원 (AWS Support 등)

온콜 도구:
├── 알람: PagerDuty, Opsgenie, VictorOps
├── 커뮤니케이션: Slack, Teams
└── Runbook: 장애 대응 절차 자동화
```

### 역량 개발

```
권장 학습 경로:

Level 1 (신입):
├── Kubernetes 기초 (CKA 수준)
├── 클러스터 기본 운영
└── 모니터링 대시보드 읽기

Level 2 (중급):
├── 네트워킹 심화 (CNI, Service Mesh)
├── 보안 (RBAC, Network Policy)
├── 트러블슈팅 (로그 분석, 디버깅)
└── 선택 솔루션 심화 (Hybrid Nodes 또는 Anywhere)

Level 3 (시니어):
├── 아키텍처 설계
├── 성능 최적화
├── 비용 최적화
├── 멀티 클러스터 운영
└── 자동화 (IaC, GitOps, CI/CD)
```

---

## 체크리스트: 프로덕션 준비도

프로덕션으로 가기 전 확인하세요:

```
아키텍처:
□ 고가용성 구성 (CP 3대, Worker 3대+)
□ 노드 물리적 분산 (랙, 전원)
□ 네트워크 이중화

보안:
□ RBAC 설정
□ Network Policy 적용
□ Pod Security Standards 적용
□ Secret 관리 방안 수립

모니터링:
□ 메트릭 수집 (Prometheus/CloudWatch)
□ 로그 수집 (Loki/CloudWatch Logs)
□ 알람 설정 (핵심 메트릭)
□ 대시보드 구성

백업:
□ etcd 백업 (EKS Anywhere)
□ Velero 설정
□ 복구 테스트 완료

운영:
□ 업그레이드 절차 문서화
□ 장애 대응 Runbook 작성
□ 온콜 체계 수립
□ 변경 관리 프로세스 수립

문서화:
□ 아키텍처 다이어그램
□ 접근 권한 문서
□ 운영 절차 문서
```

---

> ✅ **완료**
>
> **축하합니다!** 이 Best Practices를 따르면 안정적인 프로덕션 환경을 운영할 수 있습니다. 지속적으로 개선하고 팀의 피드백을 반영하세요.
