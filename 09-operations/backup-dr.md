# 백업 및 재해복구

> "클러스터가 날아가면 어떻게 하나요?"

백업 없이는 복구가 불가능합니다. 체계적인 백업 전략이 필수입니다.

## 백업 대상

```
┌─────────────────────────────────────────────────────────────────┐
│                     백업 대상                                   │
│                                                                 │
│   1. etcd (Control Plane 상태)                                 │
│      └── 클러스터 설정, 모든 리소스 정의                       │
│                                                                 │
│   2. Kubernetes 리소스                                         │
│      └── Deployments, ConfigMaps, Secrets 등                   │
│                                                                 │
│   3. Persistent Volumes                                         │
│      └── 애플리케이션 데이터                                   │
│                                                                 │
│   4. 클러스터 설정                                             │
│      └── cluster-config.yaml (EKS Anywhere)                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## Velero 설치

```bash
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.7.0 \
  --bucket my-backup-bucket \
  --backup-location-config region=ap-northeast-2 \
  --snapshot-location-config region=ap-northeast-2 \
  --secret-file ./credentials-velero
```

## 백업 예시

```yaml
# 일일 백업 스케줄
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: daily-backup
  namespace: velero
spec:
  schedule: "0 2 * * *"  # 매일 새벽 2시
  template:
    includedNamespaces:
      - production
      - staging
    excludedResources:
      - events
    ttl: 168h  # 7일 보관
    snapshotVolumes: true
```

```bash
# 수동 백업
velero backup create manual-backup --include-namespaces production

# 복구
velero restore create --from-backup manual-backup
```

## etcd 백업 (EKS Anywhere)

```bash
# etcd 스냅샷 생성
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-$(date +%Y%m%d).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# 복구
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-20240101.db
```

## DR 전략

```
┌─────────────────────────────────────────────────────────────────┐
│                    RPO/RTO 계획                                 │
│                                                                 │
│   RPO (Recovery Point Objective):                              │
│   └── 얼마나 오래된 데이터까지 복구 가능?                      │
│       예: 1시간 → 1시간마다 백업                               │
│                                                                 │
│   RTO (Recovery Time Objective):                               │
│   └── 복구에 얼마나 걸려야 하나?                               │
│       예: 4시간 → DR 훈련으로 검증                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

