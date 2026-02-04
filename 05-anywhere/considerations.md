# EKS Anywhere 주의사항과 제약

> "독립적으로 운영한다는 건, 모든 책임도 우리에게 있다는 뜻이죠?"

네, 맞습니다. EKS Anywhere의 자유도는 그에 상응하는 책임을 수반합니다. 도입 전에 알아야 할 제약사항과 주의점을 살펴봅시다.

## 핵심 제약사항

### 1. Control Plane 운영 책임

**이것이 가장 큰 차이점입니다.** etcd부터 API Server까지 모든 것을 직접 관리해야 합니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                    운영 책임 매트릭스                           │
│                                                                 │
│   항목                    책임자      예상 노력                │
│   ─────────────────────────────────────────────────────────────│
│   etcd 백업               운영팀      매일 자동화 필요         │
│   etcd 복구               운영팀      1-4시간 (장애 시)        │
│   API Server HA           운영팀      초기 구성 + 모니터링     │
│   인증서 갱신             운영팀      연 1회 (자동화 권장)     │
│   K8s 업그레이드          운영팀      분기당 1회, 2-4시간      │
│   보안 패치               운영팀      월 1-2회                 │
│   장애 대응               운영팀      24/7 온콜 필요          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**새벽 3시 시나리오**

```
알람: "etcd cluster unhealthy"
  │
  ├── 운영자 기상
  ├── VPN 접속
  ├── 상태 확인
  │   kubectl get pods -n kube-system
  │   etcdctl endpoint health
  ├── 원인 분석
  │   - 디스크 풀?
  │   - 네트워크 이슈?
  │   - 노드 장애?
  ├── 복구 작업
  │   - 장애 노드 격리
  │   - etcd 멤버 제거/추가
  │   - 필요시 백업에서 복원
  ├── 검증
  └── 약 1-4시간 소요
```

### 2. 업그레이드 복잡성

EKS Hybrid Nodes와 달리 버튼 하나로 업그레이드할 수 없습니다.

```
EKS Anywhere 업그레이드 프로세스:

┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  1. 사전 준비 (1-2시간)                                        │
│     ├── 현재 상태 백업                                         │
│     ├── 새 버전 이미지 준비                                    │
│     ├── 구성 파일 업데이트                                     │
│     └── 테스트 환경에서 검증                                   │
│                                                                 │
│  2. Control Plane 업그레이드 (30분-1시간)                      │
│     ├── eksctl anywhere upgrade cluster 실행                   │
│     ├── Rolling update 진행                                    │
│     └── 각 노드별 헬스체크                                     │
│                                                                 │
│  3. Worker Node 업그레이드 (노드당 10-15분)                    │
│     ├── Node drain                                             │
│     ├── VM 교체 또는 in-place 업그레이드                      │
│     └── Node uncordon                                          │
│                                                                 │
│  4. 검증 (30분)                                                │
│     ├── 클러스터 상태 확인                                     │
│     ├── 애플리케이션 동작 확인                                 │
│     └── 모니터링 확인                                          │
│                                                                 │
│  총 예상 시간: 3-6시간 (클러스터 규모에 따라)                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 3. AWS 서비스 통합 수동

Pod Identity, IRSA 같은 AWS 네이티브 기능이 기본 제공되지 않습니다.

```
AWS 서비스 접근 방법 비교:

EKS (Cloud) / Hybrid Nodes:
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   Pod ──► Pod Identity ──► IAM Role ──► S3/DynamoDB/etc        │
│                                                                 │
│   자동으로 임시 자격증명 획득                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

EKS Anywhere:
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   옵션 1: 수동 자격증명                                        │
│   Pod ──► Secret (Access Key) ──► AWS 서비스                   │
│           (보안 위험!)                                          │
│                                                                 │
│   옵션 2: 외부 시크릿 관리자                                   │
│   Pod ──► Vault ──► AWS STS ──► AWS 서비스                     │
│           (추가 구성 필요)                                      │
│                                                                 │
│   옵션 3: 직접 OIDC 구성                                       │
│   Pod ──► 자체 OIDC ──► AWS STS ──► AWS 서비스                │
│           (복잡한 설정)                                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 4. 지원 범위 제한

Enterprise Support 없이는 제한된 지원만 받을 수 있습니다.

```
지원 옵션:

커뮤니티 지원 (무료):
├── GitHub Issues
├── Slack 채널
├── 커뮤니티 포럼
└── SLA: 없음

Enterprise Support (유료):
├── AWS Support 채널
├── 24/7 지원
├── SLA: 있음
└── 비용: 협의 필요
```

### 5. 인프라 프로바이더 제한

모든 환경을 지원하지 않습니다.

```
지원 인프라:
├── ✅ VMware vSphere 7.0+
├── ✅ Bare Metal (Tinkerbell)
├── ✅ Nutanix
├── ✅ Apache CloudStack
├── ✅ AWS Snow
├── ❌ Hyper-V
├── ❌ KVM/Proxmox (직접 지원 없음)
├── ❌ OpenStack
└── ❌ Public Cloud (AWS/Azure/GCP)
```

## 운영 시 주의사항

### etcd 관리

etcd는 클러스터의 심장입니다. 특별한 주의가 필요합니다.

```yaml
# etcd 백업 CronJob 예시
apiVersion: batch/v1
kind: CronJob
metadata:
  name: etcd-backup
  namespace: kube-system
spec:
  schedule: "0 */6 * * *"    # 6시간마다
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: bitnami/etcd:latest
            command:
            - /bin/sh
            - -c
            - |
              etcdctl snapshot save /backup/snapshot-$(date +%Y%m%d-%H%M%S).db \
                --endpoints=https://127.0.0.1:2379 \
                --cacert=/etc/kubernetes/pki/etcd/ca.crt \
                --cert=/etc/kubernetes/pki/etcd/server.crt \
                --key=/etc/kubernetes/pki/etcd/server.key
            volumeMounts:
            - name: backup-volume
              mountPath: /backup
            - name: etcd-certs
              mountPath: /etc/kubernetes/pki/etcd
          volumes:
          - name: backup-volume
            persistentVolumeClaim:
              claimName: etcd-backup-pvc
          - name: etcd-certs
            hostPath:
              path: /etc/kubernetes/pki/etcd
          restartPolicy: OnFailure
```

### 인증서 관리

Kubernetes 인증서는 기본 1년 만료입니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                    인증서 만료 체크리스트                       │
│                                                                 │
│   체크 주기: 월 1회                                            │
│                                                                 │
│   명령어:                                                       │
│   kubeadm certs check-expiration                               │
│                                                                 │
│   주의 대상:                                                    │
│   ├── /etc/kubernetes/pki/apiserver.crt                        │
│   ├── /etc/kubernetes/pki/apiserver-kubelet-client.crt        │
│   ├── /etc/kubernetes/pki/front-proxy-client.crt              │
│   ├── /etc/kubernetes/pki/etcd/server.crt                     │
│   └── /etc/kubernetes/pki/etcd/peer.crt                       │
│                                                                 │
│   갱신 방법:                                                    │
│   kubeadm certs renew all                                      │
│                                                                 │
│   알림 설정 권장:                                               │
│   └── 만료 30일 전 알림                                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 모니터링 직접 구성

CloudWatch 통합이 없으므로 모니터링 스택을 직접 구성해야 합니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                    권장 모니터링 스택                           │
│                                                                 │
│   메트릭:                                                       │
│   ├── Prometheus (수집)                                        │
│   ├── Grafana (시각화)                                         │
│   └── AlertManager (알림)                                      │
│                                                                 │
│   로깅:                                                         │
│   ├── Fluent Bit (수집)                                        │
│   ├── Elasticsearch (저장)                                     │
│   └── Kibana (검색/시각화)                                     │
│       또는 Loki + Grafana                                      │
│                                                                 │
│   트레이싱:                                                     │
│   └── Jaeger 또는 Zipkin                                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 업그레이드 전략

```
┌─────────────────────────────────────────────────────────────────┐
│                    업그레이드 체크리스트                        │
│                                                                 │
│   사전 준비:                                                    │
│   □ 현재 클러스터 상태 확인                                    │
│   □ etcd 백업 완료                                             │
│   □ 애플리케이션 백업 (Velero)                                 │
│   □ 롤백 계획 수립                                             │
│   □ 유지보수 창 공지                                           │
│                                                                 │
│   업그레이드 실행:                                              │
│   □ Control Plane 업그레이드                                   │
│   □ Control Plane 헬스체크                                     │
│   □ Worker Node 순차 업그레이드                                │
│   □ 각 노드별 검증                                             │
│                                                                 │
│   사후 검증:                                                    │
│   □ 모든 시스템 Pod 정상                                       │
│   □ 애플리케이션 정상 동작                                     │
│   □ 모니터링 정상                                              │
│   □ 네트워크 정상                                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## 도입 전 체크리스트

### 조직 준비도

- [ ] Kubernetes 운영 경험이 있는 팀원 (최소 1명)
- [ ] 24/7 온콜 체계 (또는 수립 계획)
- [ ] Control Plane 장애 대응 역량
- [ ] etcd 백업/복구 경험

### 인프라 준비

- [ ] 지원 인프라 프로바이더 (vSphere, Bare Metal 등)
- [ ] 충분한 컴퓨팅 리소스
- [ ] 네트워크 설계 완료
- [ ] 스토리지 준비 (etcd용 SSD 권장)

### 운영 준비

- [ ] 모니터링 스택 계획
- [ ] 로깅 솔루션 계획
- [ ] 백업 전략 수립
- [ ] 업그레이드 절차 문서화
- [ ] 장애 대응 런북 작성

### 비용 계획

- [ ] Enterprise Support 필요 여부 검토
- [ ] 운영 인력 비용 산정
- [ ] 교육 비용 계획

## 적합하지 않은 경우

| 상황 | 이유 | 대안 |
|------|------|------|
| 운영 인력 부족 | Control Plane 관리 부담 | EKS Hybrid Nodes |
| K8s 경험 부족 | 학습 곡선 높음 | EKS (Cloud) 시작 |
| AWS 서비스 의존도 높음 | 통합 복잡 | EKS Hybrid Nodes |
| 빠른 도입 필요 | 구성 시간 필요 | EKS (Cloud) |
| SLA 필수 | 기본 SLA 없음 | EKS Hybrid Nodes |

---

다음 장에서는 Multi-Cluster 운영에 대해 알아봅니다.
