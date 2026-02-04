# CNI 선택: Cilium vs Calico

> "CNI가 뭐고, 어떤 걸 써야 하나요?"

CNI(Container Network Interface)는 Pod 네트워킹을 담당합니다. 온프레미스 EKS에서는 Cilium이 주로 사용됩니다.

## CNI 비교

| 기능 | Cilium | Calico |
|------|--------|--------|
| **기반 기술** | eBPF | iptables/eBPF |
| **성능** | 매우 높음 | 높음 |
| **Network Policy** | L3/L4/L7 | L3/L4 |
| **암호화** | WireGuard | WireGuard/IPsec |
| **관측성** | Hubble (우수) | 기본 |
| **Service Mesh** | 내장 | 별도 |
| **EKS 지원** | ✅ | ✅ |
| **학습 곡선** | 중간 | 낮음 |

## Cilium 장점

```
┌─────────────────────────────────────────────────────────────────┐
│                    Cilium의 eBPF 기반 장점                      │
│                                                                 │
│   기존 방식 (iptables):                                        │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  Packet → iptables rules (수천 개) → 처리                │  │
│   │                                                          │  │
│   │  • 규칙 많아지면 성능 저하                               │  │
│   │  • 디버깅 어려움                                         │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                 │
│   Cilium (eBPF):                                               │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  Packet → eBPF program (커널 내) → 처리                  │  │
│   │                                                          │  │
│   │  • 일관된 높은 성능                                      │  │
│   │  • Hubble로 뛰어난 관측성                                │  │
│   │  • L7 정책 지원                                          │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## Cilium 설치

```yaml
# values.yaml
cluster:
  name: my-cluster
  id: 1

ipam:
  mode: cluster-pool
  operator:
    clusterPoolIPv4PodCIDRList:
      - 100.64.0.0/16
    clusterPoolIPv4MaskSize: 24

hubble:
  enabled: true
  relay:
    enabled: true
  ui:
    enabled: true

encryption:
  enabled: true
  type: wireguard
```

```bash
helm install cilium cilium/cilium \
  --namespace kube-system \
  -f values.yaml
```

## Hubble (관측성)

```bash
# Hubble CLI로 트래픽 모니터링
hubble observe --namespace production

# 특정 Pod 트래픽
hubble observe --pod production/frontend

# HTTP 요청만
hubble observe -t l7 --protocol http
```

## 권장사항

```
EKS Hybrid Nodes:
└── Cilium 권장 (AWS 공식 지원)

EKS Anywhere:
└── Cilium 기본 (변경 불필요)
```

