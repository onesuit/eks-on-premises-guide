# 모니터링 스택 구성

> "클러스터 상태를 어떻게 확인하나요?"

온프레미스 환경에서는 모니터링 스택을 직접 구성해야 합니다.

## 모니터링 스택

```
┌─────────────────────────────────────────────────────────────────┐
│                    표준 모니터링 스택                           │
│                                                                 │
│   수집          저장          시각화        알림              │
│   ┌──────┐     ┌──────┐     ┌──────┐     ┌──────────┐        │
│   │Prome-│────►│Prome-│────►│Grafa-│────►│AlertMana-│        │
│   │theus │     │theus │     │na    │     │ger       │        │
│   └──────┘     └──────┘     └──────┘     └──────────┘        │
│                                                │               │
│                                                ▼               │
│                                          Slack, PagerDuty     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## Kube-Prometheus-Stack 설치

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  -f values.yaml
```

```yaml
# values.yaml
grafana:
  adminPassword: "your-password"
  persistence:
    enabled: true
    size: 10Gi

prometheus:
  prometheusSpec:
    retention: 15d
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: standard
          resources:
            requests:
              storage: 50Gi

alertmanager:
  config:
    receivers:
    - name: 'slack'
      slack_configs:
      - api_url: 'https://hooks.slack.com/...'
        channel: '#alerts'
```

## 핵심 메트릭

```
노드 레벨:
├── CPU 사용률
├── 메모리 사용률
├── 디스크 사용률
└── 네트워크 I/O

Pod 레벨:
├── CPU/Memory requests vs usage
├── 재시작 횟수
├── OOMKill 발생
└── Pending 상태

클러스터 레벨:
├── API Server 레이턴시
├── etcd 상태
├── 스케줄링 실패
└── 노드 상태
```

## EKS Hybrid Nodes: CloudWatch 통합

```yaml
# CloudWatch Agent ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: cloudwatch-config
data:
  cwagentconfig.json: |
    {
      "logs": {
        "metrics_collected": {
          "kubernetes": {
            "cluster_name": "my-cluster",
            "metrics_collection_interval": 60
          }
        }
      }
    }
```

