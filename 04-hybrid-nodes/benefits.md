# EKS Hybrid Nodes 장점과 활용 사례

> "그래서 Hybrid Nodes를 쓰면 뭐가 좋은 거예요?"

EKS Hybrid Nodes의 실질적인 장점과 어떤 상황에서 가장 빛나는지 알아봅시다.

## 핵심 장점

### 1. Control Plane 운영 부담 제거

직접 K8s를 운영할 때와 비교해봅시다:

**직접 운영 시 (새벽 3시 시나리오)**

```
알람: "etcd 리더 선출 실패"
  │
  ├── 운영자 기상
  ├── VPN 접속
  ├── etcd 클러스터 상태 확인
  ├── 로그 분석
  ├── 장애 노드 격리
  ├── etcd 복구 작업
  ├── 클러스터 상태 검증
  └── 약 2-3시간 소요...
```

**EKS Hybrid Nodes 사용 시**

```
알람: (발생하지 않음)
  │
  └── Control Plane은 AWS가 관리
      • 자동 복구
      • 자동 스케일링
      • 99.95% SLA
```

| 운영 항목 | 직접 운영 | Hybrid Nodes |
|----------|----------|--------------|
| etcd 백업 | 매일 수동 또는 자동화 필요 | AWS 자동 |
| etcd 복구 | 직접 수행 (1-2시간) | AWS 자동 (분 단위) |
| API Server HA | Load Balancer 구성 필요 | 기본 제공 |
| 업그레이드 | 2-4시간 다운타임 위험 | 버튼 클릭 |
| 인증서 갱신 | 1년마다 수동 | 자동 |
| 보안 패치 | CVE 모니터링 + 적용 | AWS |

### 2. AWS 서비스 네이티브 통합

온프레미스에서 실행되는 Pod도 AWS 서비스를 쉽게 사용할 수 있습니다.

```yaml
# 온프레미스 Pod에서 S3 접근 예시
apiVersion: v1
kind: Pod
metadata:
  name: data-processor
spec:
  serviceAccountName: s3-access-sa    # Pod Identity 사용
  nodeSelector:
    eks.amazonaws.com/compute-type: hybrid
  containers:
  - name: processor
    image: my-processor:latest
    env:
    - name: S3_BUCKET
      value: my-data-bucket
    # AWS SDK가 자동으로 IAM 역할 사용
```

**통합 가능한 AWS 서비스:**

| 서비스 | 활용 예시 |
|--------|----------|
| **S3** | 온프레미스 처리 결과를 클라우드에 저장 |
| **DynamoDB** | 세션 스토어, 캐시 |
| **SQS/SNS** | 메시지 큐, 알림 |
| **Secrets Manager** | 중앙화된 시크릿 관리 |
| **CloudWatch** | 로그 및 메트릭 통합 |
| **RDS** | 데이터베이스 연결 |
| **Lambda** | 이벤트 트리거 |

### 3. 통합된 관리 경험

하나의 도구, 하나의 워크플로우:

```bash
# 클라우드와 온프레미스 노드를 동일하게 관리
kubectl get nodes
NAME                          STATUS   ROLES    AGE   VERSION   LABELS
ip-10-0-1-100.ec2.internal   Ready    <none>   5d    v1.31     eks.amazonaws.com/compute-type=ec2
ip-10-0-2-101.ec2.internal   Ready    <none>   5d    v1.31     eks.amazonaws.com/compute-type=ec2
hybrid-dc1-001               Ready    <none>   2d    v1.31     eks.amazonaws.com/compute-type=hybrid
hybrid-dc1-002               Ready    <none>   2d    v1.31     eks.amazonaws.com/compute-type=hybrid

# 동일한 방식으로 워크로드 배포
kubectl apply -f deployment.yaml

# 동일한 방식으로 로그 확인
kubectl logs -f deployment/my-app
```

### 4. 점진적 마이그레이션

클라우드와 온프레미스를 유연하게 오갈 수 있습니다:

```yaml
# Phase 1: 온프레미스에서 시작
spec:
  nodeSelector:
    eks.amazonaws.com/compute-type: hybrid

# Phase 2: 하이브리드 (둘 다)
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 50
        preference:
          matchExpressions:
          - key: eks.amazonaws.com/compute-type
            operator: In
            values: [hybrid]

# Phase 3: 클라우드로 완전 이전
spec:
  nodeSelector:
    eks.amazonaws.com/compute-type: ec2
```

## 활용 사례

### 사례 1: 데이터 레이턴시가 중요한 경우

**상황**: 제조 공장에서 센서 데이터를 실시간으로 처리해야 함

```
┌─────────────────────────────────────────────────────────────────┐
│                      Factory Floor                               │
│                                                                 │
│   센서 ──► Edge Gateway ──► Hybrid Node                         │
│                              │                                  │
│                              ▼                                  │
│                        ┌──────────┐                            │
│                        │ 실시간   │                            │
│                        │ 데이터   │ ← 5ms 이내 처리            │
│                        │ 처리 Pod │                            │
│                        └────┬─────┘                            │
│                             │                                  │
└─────────────────────────────┼───────────────────────────────────┘
                              │
                         결과만 전송
                              │
                              ▼
                    ┌───────────────────┐
                    │    AWS Cloud      │
                    │  (분석, 저장)     │
                    └───────────────────┘
```

**구성:**
- 실시간 처리는 온프레미스 Hybrid Node에서
- 집계 데이터만 클라우드로 전송
- 동일한 K8s 클러스터에서 관리

### 사례 2: 규정 준수가 필요한 경우

**상황**: 금융 데이터는 온프레미스에 저장해야 하지만, 관리는 효율화하고 싶음

```
┌─────────────────────────────────────────────────────────────────┐
│                    EKS Cluster                                   │
│                                                                 │
│   ┌─────────────────────┐    ┌─────────────────────────────┐   │
│   │   AWS (Cloud)       │    │   On-Premises (Hybrid)      │   │
│   │                     │    │                             │   │
│   │  ┌───────────────┐  │    │  ┌───────────────────────┐ │   │
│   │  │ 웹 프론트엔드 │  │    │  │ 금융 데이터 처리      │ │   │
│   │  │ (Public)      │  │    │  │ (Sensitive Data)      │ │   │
│   │  └───────┬───────┘  │    │  └───────────┬───────────┘ │   │
│   │          │          │    │              │             │   │
│   │          │          │    │  ┌───────────▼───────────┐ │   │
│   │          │          │    │  │ 온프레미스 DB         │ │   │
│   │          │          │    │  │ (규정 준수)           │ │   │
│   │          │          │    │  └───────────────────────┘ │   │
│   │          │          │    │                             │   │
│   └──────────┼──────────┘    └─────────────────────────────┘   │
│              │                             ▲                    │
│              │         Internal Service    │                    │
│              └─────────────────────────────┘                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**구성:**
- 민감 데이터 처리는 온프레미스에서
- 웹 프론트엔드는 클라우드에서
- 내부 Service로 통신

### 사례 3: 기존 인프라 활용

**상황**: 감가상각이 남은 서버가 많고, 이를 효율적으로 활용하고 싶음

```
Before:
┌────────────────────────────────────────────────────────────────┐
│  데이터센터                                                    │
│                                                                │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ │
│  │Server 1 │ │Server 2 │ │Server 3 │ │Server 4 │ │Server 5 │ │
│  │(VM)     │ │(VM)     │ │(VM)     │ │(VM)     │ │(VM)     │ │
│  │ 30%     │ │ 25%     │ │ 40%     │ │ 20%     │ │ 35%     │ │
│  │활용률   │ │활용률   │ │활용률   │ │활용률   │ │활용률   │ │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────┘ │
│                                                                │
│  평균 활용률: 30%                                              │
└────────────────────────────────────────────────────────────────┘

After (Hybrid Nodes):
┌────────────────────────────────────────────────────────────────┐
│  데이터센터                                                    │
│                                                                │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ │
│  │Hybrid 1 │ │Hybrid 2 │ │Hybrid 3 │ │Hybrid 4 │ │Hybrid 5 │ │
│  │(K8s)    │ │(K8s)    │ │(K8s)    │ │(K8s)    │ │(K8s)    │ │
│  │ 70%     │ │ 75%     │ │ 80%     │ │ 65%     │ │ 70%     │ │
│  │활용률   │ │활용률   │ │활용률   │ │활용률   │ │활용률   │ │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────┘ │
│                                                                │
│  평균 활용률: 72%  ← 컨테이너 빈 패킹으로 효율 증가            │
└────────────────────────────────────────────────────────────────┘
```

### 사례 4: 버스팅 (Bursting)

**상황**: 평소에는 온프레미스로 충분하지만, 피크 시에는 추가 용량 필요

```
                    Normal Load                    Peak Load
                   ┌───────────┐               ┌───────────┐
     Capacity      │           │               │           │
         ▲         │   ████    │               │   ████████│
         │         │   ████    │               │   ████████│
    Cloud│         │   ████    │               │   █ Cloud █│
         │         │           │               │   █ Burst █│
         │─────────│───────────│───────────────│───████████│──
         │         │   ████    │               │   ████████│
         │         │   ████    │               │   ████████│
   On-Prem         │   ████    │               │   ████████│
         │         │   ████    │               │   ████████│
         │         │   ████    │               │   ████████│
         └─────────└───────────┘               └───────────┘
                    (온프레미스만)              (온프레미스 + 클라우드)
```

```yaml
# HPA와 결합
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 5     # 온프레미스에서 최소 유지
  maxReplicas: 50    # 클라우드로 버스팅 가능
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
---
# 온프레미스 우선, 부족하면 클라우드
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  template:
    spec:
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            preference:
              matchExpressions:
              - key: eks.amazonaws.com/compute-type
                operator: In
                values: [hybrid]
```

## ROI 분석

### 정량적 이점

| 항목 | 직접 K8s 운영 | Hybrid Nodes | 절감 |
|------|-------------|--------------|------|
| Control Plane 서버 | 3-5대 | 0대 | 하드웨어 비용 |
| 운영 인력 (FTE) | 0.5-1 | 0.1-0.2 | 인건비 60-80% |
| 업그레이드 다운타임 | 2-4시간/분기 | 0 | 가용성 향상 |
| 장애 복구 시간 | 1-4시간 | 분 단위 | 비즈니스 연속성 |

### 정성적 이점

```
✅ 운영팀이 인프라 대신 애플리케이션에 집중
✅ 클라우드 팀과 동일한 스킬셋 사용
✅ AWS 에코시스템 활용 (모니터링, 보안, CI/CD)
✅ 점진적 클라우드 마이그레이션 경로 확보
```

