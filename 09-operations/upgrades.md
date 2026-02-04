# 업그레이드 전략

> "업그레이드가 무서워요. 어떻게 안전하게 하나요?"

업그레이드는 피할 수 없습니다. 체계적인 전략으로 위험을 최소화합니다.

## 업그레이드 원칙

```
┌─────────────────────────────────────────────────────────────────┐
│                    업그레이드 황금 규칙                         │
│                                                                 │
│   1. 한 번에 하나씩                                            │
│      └── 한 마이너 버전씩만 업그레이드                         │
│                                                                 │
│   2. 테스트 먼저                                                │
│      └── Dev → Staging → Prod 순서                             │
│                                                                 │
│   3. 백업 필수                                                  │
│      └── 업그레이드 전 etcd, 워크로드 백업                     │
│                                                                 │
│   4. 롤백 계획                                                  │
│      └── 실패 시 복구 절차 준비                                │
│                                                                 │
│   5. 유지보수 창                                                │
│      └── 사전 공지, 낮은 트래픽 시간대                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## EKS Hybrid Nodes 업그레이드

```
순서:

1. Control Plane 업그레이드 (AWS 콘솔/CLI)
   └── aws eks update-cluster-version ...
   └── 몇 분 소요, 자동 처리

2. Add-ons 업그레이드
   └── aws eks update-addon ...

3. Hybrid Nodes 업그레이드
   └── 각 노드에서:
       kubectl drain node-1 --ignore-daemonsets
       sudo nodeadm upgrade
       kubectl uncordon node-1
```

## EKS Anywhere 업그레이드

```
순서:

1. 클러스터 설정 업데이트
   └── kubernetesVersion: "1.31"

2. 업그레이드 실행
   └── eksctl anywhere upgrade cluster -f cluster.yaml

3. 상태 확인
   └── kubectl get nodes
   └── kubectl get pods -n kube-system
```

## 버전 호환성

```
Control Plane v1.31
├── Worker Node v1.31 ✅ (동일)
├── Worker Node v1.30 ✅ (n-1)
├── Worker Node v1.29 ✅ (n-2)
└── Worker Node v1.28 ❌ (n-3, 미지원)

항상 Control Plane을 먼저 업그레이드!
```

---

[다음: 모니터링 스택 →](monitoring.md)
