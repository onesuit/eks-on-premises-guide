# Control Plane 깊이 알아보기

> "Kubernetes가 알아서 해준다고 하는데, 정확히 누가 뭘 하는 건가요?"

Control Plane은 Kubernetes 클러스터의 "두뇌"입니다. 직접 컨테이너를 실행하지는 않지만, 모든 결정을 내립니다.

> 💡 **참고**
>
> **이미 EKS를 알고 계신가요?**
>
> EKS 경험이 있다면 이 섹션의 기본 개념 부분은 건너뛰고, 아래의 "Control Plane 관리, 왜 부담인가?" 섹션부터 읽으셔도 됩니다.

## Control Plane 구성 요소

```
┌────────────────────────────────────────────────────────────────┐
│                       Control Plane                             │
│                                                                 │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────────────────┐   │
│  │ API Server  │  │ Scheduler   │  │ Controller Manager   │   │
│  │             │  │             │  │                      │   │
│  │ "접수 창구" │  │ "배치 담당" │  │ "상태 유지 담당"     │   │
│  └──────┬──────┘  └──────┬──────┘  └──────────┬───────────┘   │
│         │                │                     │               │
│         └────────────────┼─────────────────────┘               │
│                          │                                     │
│                          ▼                                     │
│                   ┌─────────────┐                              │
│                   │    etcd     │                              │
│                   │             │                              │
│                   │ "기억 저장소"│                              │
│                   └─────────────┘                              │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

## 각 구성 요소의 역할

### 1. API Server: 모든 것의 시작점

API Server는 Kubernetes의 "프론트 데스크"입니다. `kubectl` 명령어부터 내부 컴포넌트 간 통신까지, 모든 요청은 API Server를 거칩니다.

**실제로 하는 일:**
- 사용자가 `kubectl apply -f deployment.yaml`을 실행하면 이 요청을 받음
- "이 사용자가 이 작업을 할 권한이 있는지" 확인 (인증/인가)
- 요청이 유효한지 검증 (YAML 문법, 필수 필드 등)
- etcd에 저장하고 다른 컴포넌트에 알림

```bash
# kubectl은 사실 API Server에 HTTP 요청을 보내는 것
kubectl get pods

# 위 명령은 내부적으로 이렇게 동작:
curl -k https://<api-server>:6443/api/v1/namespaces/default/pods \
  -H "Authorization: Bearer <token>"
```

### 2. etcd: 클러스터의 기억

etcd는 분산 키-값 저장소입니다. "클러스터에 어떤 Deployment가 있고, 어떤 Pod가 실행 중인지" 등 모든 상태 정보가 여기에 저장됩니다.

**왜 중요한가요?**

etcd가 없으면 Kubernetes는 "기억상실"에 걸립니다. 어떤 Pod를 실행해야 하는지, 어떤 Service가 있는지 전혀 모르게 됩니다. 그래서 etcd 관리가 매우 중요하고, 이것이 바로 Control Plane 관리의 핵심 부담 중 하나입니다.

```
etcd에 저장되는 것들:
├── 모든 Pod, Deployment, Service 정보
├── ConfigMap, Secret 내용
├── RBAC 정책 (누가 무엇을 할 수 있는지)
├── 현재 클러스터 상태
└── 사실상 kubectl로 보는 모든 것
```

### 3. Controller Manager: "원하는 상태"를 유지하는 관리자

Controller Manager는 여러 개의 Controller를 실행합니다. 각 Controller는 특정 리소스 타입을 담당하며, "선언한 상태"와 "실제 상태"를 맞추는 역할을 합니다.

**왜 Controller라고 부르나요?**

"에어컨 컨트롤러"를 생각해보세요. 여러분이 "25도로 맞춰줘"라고 설정하면, 컨트롤러가 현재 온도를 측정하고, 25도가 될 때까지 냉방이나 난방을 작동시킵니다. Kubernetes Controller도 마찬가지입니다.

```
여러분: "nginx Pod 3개 실행해줘" (Deployment 생성)

Deployment Controller의 동작:
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  원하는 상태: replicas = 3                                     │
│  현재 상태:   실행 중 = 2                                      │
│                                                                 │
│  Controller 판단: "1개 부족하네! Pod 하나 더 만들자"           │
│                                                                 │
│  → ReplicaSet Controller에게 Pod 생성 요청                     │
│  → Scheduler가 어느 노드에 배치할지 결정                       │
│  → kubelet이 실제로 컨테이너 시작                              │
│                                                                 │
│  현재 상태:   실행 중 = 3 ✓                                    │
│  Controller: "OK, 원하는 상태와 일치!"                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**주요 Controller들:**

| Controller | 하는 일 |
|------------|--------|
| **Deployment Controller** | "nginx 3개 유지해줘" 같은 요청을 처리. ReplicaSet을 만들어서 Pod 수 관리 |
| **ReplicaSet Controller** | 실제로 Pod 개수를 유지. Pod가 죽으면 새로 생성 |
| **Node Controller** | 노드 상태를 감시. 노드가 응답이 없으면 그 위의 Pod를 다른 노드로 이동 |
| **Service Controller** | LoadBalancer 타입 Service를 만들면 실제 로드밸런서 생성 요청 |

### 4. Scheduler: 어디에 배치할까?

새 Pod가 생성되면 Scheduler가 "어떤 노드에서 실행할지" 결정합니다.

**비유**: 회사에 새 직원이 오면 어느 팀에 배치할지 결정하는 인사팀과 비슷합니다.

```
Scheduler의 판단 과정:

1단계 - 필터링 (가능한 노드 추리기):
├── Node 1: CPU 부족 ❌ (탈락)
├── Node 2: 메모리 충분 ✓
├── Node 3: "GPU 전용" 라벨인데 이 Pod는 GPU 안 씀 ❌ (탈락)
└── Node 4: 조건 충족 ✓

2단계 - 스코어링 (최적 노드 선택):
├── Node 2: 점수 75 (이미 Pod가 많음)
└── Node 4: 점수 82 (여유 있음) ← 선택!
```

---

## Control Plane 관리, 왜 부담인가?

여기서부터가 핵심입니다. "왜 Control Plane 관리를 AWS에게 맡기는 것이 장점인가?"를 이해하려면, 직접 운영할 때 어떤 작업들을 해야 하는지 알아야 합니다.

### 직접 운영 시 해야 하는 작업들

#### 1. etcd 관리 (가장 중요하고 어려움)

etcd는 클러스터의 모든 데이터를 저장합니다. 이것이 날아가면 클러스터 전체를 처음부터 다시 구축해야 합니다.

**직접 운영 시 해야 하는 것들:**

```
매일:
├── 자동 백업 확인 (실제로 백업이 되고 있는지)
├── 백업 무결성 검증 (백업 파일이 손상되지 않았는지)
└── 디스크 용량 모니터링 (etcd는 디스크가 차면 작동 중지)

주기적으로:
├── 성능 모니터링 (레이턴시가 높아지면 클러스터 전체 느려짐)
├── 압축(compaction) 실행 (히스토리 데이터 정리)
└── 조각 모음(defragmentation) 실행

장애 시:
├── 멤버 교체 (노드 죽으면 새 노드로 교체)
├── 스냅샷에서 복구
└── 클러스터 재구성
```

**실제 백업 명령어:**

```bash
# etcd 스냅샷 백업 (직접 운영 시 이걸 자동화해야 함)
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-$(date +%Y%m%d-%H%M).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# 백업 검증
ETCDCTL_API=3 etcdctl snapshot status /backup/etcd-20240101-0200.db
```

#### 2. 인증서 관리

Kubernetes 컴포넌트들은 TLS 인증서로 서로를 인증합니다. 이 인증서들은 **기본 1년** 후에 만료됩니다.

```
만료되면 생기는 일:
├── API Server 접속 불가
├── kubectl 명령어 작동 안 함
├── Pod 스케줄링 중지
├── 모든 것이 멈춤!

관리해야 하는 인증서들:
├── API Server 인증서
├── kubelet 인증서
├── etcd 인증서 (서버, 피어, 클라이언트)
├── Controller Manager 인증서
├── Scheduler 인증서
├── Front Proxy 인증서
└── Service Account 키
```

**직접 갱신 시:**

```bash
# 인증서 만료일 확인
kubeadm certs check-expiration

# 모든 인증서 갱신 (클러스터 재시작 필요할 수 있음)
kubeadm certs renew all
```

#### 3. 업그레이드

Kubernetes는 약 4개월마다 새 버전이 나옵니다. 보안 패치와 버그 수정을 위해 정기적인 업그레이드가 필요합니다.

**직접 업그레이드 절차:**

```
사전 준비 (1-2시간):
├── 변경 사항 문서 읽기 (Deprecated API 확인)
├── 현재 클러스터 백업
├── 업그레이드 계획 수립
└── 롤백 계획 준비

Control Plane 업그레이드 (30분-1시간):
├── 첫 번째 Master 노드 업그레이드
│   ├── kubeadm upgrade plan
│   ├── kubeadm upgrade apply v1.31.0
│   └── kubelet, kubectl 업그레이드
├── 두 번째, 세 번째 Master 노드 동일 작업
└── 각 단계마다 상태 확인

Worker 노드 업그레이드 (노드당 10-15분):
├── kubectl drain node-1 --ignore-daemonsets
├── apt-get upgrade kubelet kubectl
├── systemctl restart kubelet
└── kubectl uncordon node-1

사후 검증 (30분):
├── 모든 시스템 Pod 정상 확인
├── 애플리케이션 동작 확인
└── 모니터링 확인
```

#### 4. 고가용성(HA) 구성 및 유지

프로덕션에서는 Control Plane이 단일 장애점(SPOF)이 되면 안 됩니다.

```
HA 구성을 위해 해야 하는 것:
├── Master 노드 3대 이상 준비
├── etcd 클러스터 구성 (3대 이상, 홀수)
├── API Server 앞에 로드밸런서 구성
├── Leader Election 설정 확인
└── 장애 테스트 (한 노드 죽여보고 복구되는지 확인)
```

#### 5. 모니터링 및 알림

Control Plane이 죽으면 Pod는 계속 실행되지만, 새로운 Pod 배포, 스케일링, 장애 복구가 모두 멈춥니다.

```
모니터링해야 하는 것들:
├── API Server
│   ├── 응답 시간 (느려지면 모든 작업 지연)
│   ├── 에러율
│   └── 요청 수
├── etcd
│   ├── 리더 상태
│   ├── 디스크 사용량
│   ├── 레이턴시
│   └── 멤버 상태
├── Scheduler
│   └── 스케줄링 지연 시간
└── Controller Manager
    └── 동기화 지연 시간
```

### 새벽 3시 시나리오: 직접 운영 vs EKS

**직접 운영 시:**

```
새벽 3시 알람: "etcd 리더 선출 실패"

03:00 - 알람 수신, 기상
03:05 - VPN 접속
03:10 - SSH로 Master 노드 접속
03:15 - etcd 상태 확인
       $ ETCDCTL_API=3 etcdctl endpoint status
       $ ETCDCTL_API=3 etcdctl endpoint health
03:25 - 원인 분석 (디스크 풀? 네트워크? OOM?)
03:40 - 문제 노드 식별
03:50 - etcd 멤버 제거 시도
       $ ETCDCTL_API=3 etcdctl member remove <member-id>
04:00 - 새 노드에 etcd 설치 및 클러스터 조인
04:30 - 클러스터 상태 복구 확인
04:45 - API Server 정상 동작 확인
05:00 - 포스트모템 작성 시작
05:30 - 다시 잠자리에...

총 소요 시간: 2-3시간
정신적 피로도: 극심
```

**EKS (또는 Hybrid Nodes) 사용 시:**

```
새벽 3시: (알람 없음)
       
AWS가 자동으로:
├── 이상 감지
├── 자동 복구 수행
├── 필요시 노드 교체
└── 99.95% SLA 보장

당신: 😴 (계속 수면)

아침에 확인:
└── CloudWatch에서 "minor incident recovered automatically" 확인
```

---

## EKS, Hybrid Nodes, Anywhere에서의 Control Plane 관리 비교

| 작업 | 직접 운영 (kubeadm) | EKS (Cloud) | EKS Hybrid Nodes | EKS Anywhere |
|------|-------------------|-------------|------------------|--------------|
| **etcd 백업** | 직접 구성, 매일 확인 | AWS 자동 | AWS 자동 | 직접 구성 |
| **etcd 복구** | 직접 수행 (1-4시간) | AWS 자동 (분 단위) | AWS 자동 | 직접 수행 |
| **인증서 갱신** | 매년 수동 | 자동 | 자동 | 직접 관리 |
| **K8s 업그레이드** | 2-4시간, 다운타임 위험 | 버튼 클릭, 자동 롤링 | 버튼 클릭 | 직접 수행 |
| **HA 구성** | 직접 구성 | 기본 제공 | 기본 제공 | 직접 구성 |
| **보안 패치** | CVE 모니터링 + 수동 적용 | AWS가 적용 | AWS가 적용 | 직접 적용 |
| **장애 대응** | 24/7 온콜 필요 | AWS Support | AWS Support | 직접 대응 |
| **SLA** | 없음 (자체 책임) | 99.95% | 99.95% | 없음 |

> ✅ **완료**
>
> **핵심 포인트**
>
> EKS와 Hybrid Nodes를 사용하면 Control Plane 운영의 가장 어렵고 리스크가 큰 부분(etcd 관리, 인증서 갱신, 업그레이드)을 AWS가 대신 처리해줍니다. 
>
> 이것이 "Control Plane 관리 부담을 줄인다"의 실제 의미입니다.

