# 방화벽 규칙 설정

> "어떤 포트를 열어야 하나요?"

온프레미스 방화벽 설정이 잘못되면 클러스터가 동작하지 않습니다.

## EKS Hybrid Nodes 필수 포트

```
┌─────────────────────────────────────────────────────────────────┐
│              On-Prem → AWS (Outbound) 필수                      │
│                                                                 │
│   대상                              포트      프로토콜         │
│   ─────────────────────────────────────────────────────────────│
│   EKS API Server                    443       TCP              │
│   SSM Endpoints                     443       TCP              │
│   ECR Endpoints                     443       TCP              │
│   S3 (이미지 레이어)               443       TCP              │
│   CloudWatch (선택)                 443       TCP              │
│   STS (IAM RA 사용 시)             443       TCP              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│              On-Prem 내부 (Node 간) 필수                        │
│                                                                 │
│   용도                              포트      프로토콜         │
│   ─────────────────────────────────────────────────────────────│
│   Kubelet API                       10250     TCP              │
│   Cilium Health                     4240      TCP              │
│   Cilium VXLAN                      8472      UDP              │
│   WireGuard (암호화)                51871     UDP              │
│   NodePort 범위                     30000-32767 TCP/UDP        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## EKS Anywhere 필수 포트

```
┌─────────────────────────────────────────────────────────────────┐
│              Control Plane 노드 필수                            │
│                                                                 │
│   용도                              포트      프로토콜         │
│   ─────────────────────────────────────────────────────────────│
│   Kubernetes API                    6443      TCP              │
│   etcd                              2379-2380 TCP              │
│   Kubelet                           10250     TCP              │
│   Scheduler                         10259     TCP              │
│   Controller Manager                10257     TCP              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│              Worker 노드 필수                                   │
│                                                                 │
│   용도                              포트      프로토콜         │
│   ─────────────────────────────────────────────────────────────│
│   Kubelet API                       10250     TCP              │
│   NodePort 범위                     30000-32767 TCP/UDP        │
│   Cilium                            4240, 8472 TCP/UDP         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## 방화벽 규칙 예시 (iptables)

```bash
# EKS API Server로 아웃바운드
iptables -A OUTPUT -p tcp -d eks.ap-northeast-2.amazonaws.com --dport 443 -j ACCEPT

# 노드 간 통신
iptables -A INPUT -p tcp --dport 10250 -s 192.168.1.0/24 -j ACCEPT

# Cilium VXLAN
iptables -A INPUT -p udp --dport 8472 -s 192.168.1.0/24 -j ACCEPT
```

## 체크리스트

```
┌─────────────────────────────────────────────────────────────────┐
│                 네트워크 연결 테스트                            │
│                                                                 │
│   EKS Hybrid Nodes:                                            │
│   □ curl -v https://eks.{region}.amazonaws.com                 │
│   □ curl -v https://ssm.{region}.amazonaws.com                 │
│   □ curl -v https://ecr.{region}.amazonaws.com                 │
│                                                                 │
│   노드 간:                                                      │
│   □ nc -zv {other-node-ip} 10250                               │
│   □ nc -zv {other-node-ip} 8472                                │
│                                                                 │
│   DNS:                                                          │
│   □ nslookup kubernetes.default.svc.cluster.local              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

