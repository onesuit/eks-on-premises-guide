# EKS Anywhere 시작하기 체크리스트

> EKS Anywhere를 시작하기 위한 단계별 체크리스트입니다. EKS Anywhere는 Control Plane까지 직접 관리하므로 더 많은 준비가 필요합니다.

---

## Phase 1: 사전 준비

### 인프라 요구사항 확인

```
□ 지원 인프라 유형 확인
  ├── VMware vSphere 7.0+
  ├── Bare Metal (Tinkerbell)
  ├── CloudStack
  ├── Nutanix
  └── Snow (AWS Outposts)

□ 하드웨어 요구사항 확인
```

**최소 사양 (개발/테스트):**

| 역할 | 수량 | CPU | RAM | 스토리지 |
|------|------|-----|-----|----------|
| Admin Machine | 1 | 4 | 16GB | 100GB |
| Control Plane | 1 | 2 | 8GB | 100GB |
| Worker Node | 1 | 2 | 8GB | 100GB |

**권장 사양 (프로덕션):**

| 역할 | 수량 | CPU | RAM | 스토리지 |
|------|------|-----|-----|----------|
| Admin Machine | 1 | 4 | 16GB | 200GB |
| Control Plane | 3 | 4 | 16GB | 200GB |
| Worker Node | 3+ | 8 | 32GB | 500GB |

### Admin Machine 설정

```bash
□ 지원 OS 확인 (Admin Machine)
  └── Ubuntu 20.04/22.04 권장

□ Docker 설치
$ sudo apt-get update
$ sudo apt-get install -y docker.io
$ sudo usermod -aG docker $USER
$ newgrp docker
$ docker --version

□ kubectl 설치
$ curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
$ sudo install kubectl /usr/local/bin/kubectl
$ kubectl version --client

□ eksctl-anywhere 설치
$ curl -fsSL "https://anywhere.eks.amazonaws.com/releases/eks-a/latest/download/eksctl-anywhere-linux-amd64.tar.gz" | tar xz -C /tmp
$ sudo mv /tmp/eksctl-anywhere /usr/local/bin/
$ eksctl anywhere version
```

### 네트워크 준비

```
□ IP 대역 계획
  ├── Management Network CIDR
  ├── Control Plane Endpoint IP
  ├── Pod CIDR (기본: 192.168.0.0/16)
  └── Service CIDR (기본: 10.96.0.0/12)

□ DHCP 또는 고정 IP 결정
  └── Control Plane은 고정 IP 권장

□ DNS 서버 확인
□ NTP 서버 확인 (시간 동기화 중요!)
□ 인터넷 접근 여부 확인 (에어갭 환경인지)
```

---

## Phase 2: 인프라별 설정

### VMware vSphere 환경

```
□ vSphere 버전 확인 (7.0 이상)
□ vCenter Server 접근 정보
  ├── vCenter Server 주소
  ├── 사용자명/비밀번호
  └── 데이터센터 이름

□ 필요 권한 확인
  ├── VM 생성/삭제
  ├── 네트워크 설정
  └── 스토리지 접근

□ OVA 템플릿 업로드
$ eksctl anywhere download artifacts \
  --provider vsphere \
  -o /tmp/artifacts

# vCenter에 OVA 업로드
```

**vSphere 클러스터 설정 파일:**

```yaml
# vsphere-cluster.yaml
apiVersion: anywhere.eks.amazonaws.com/v1alpha1
kind: Cluster
metadata:
  name: my-eks-anywhere
spec:
  clusterNetwork:
    cniConfig:
      cilium: {}
    pods:
      cidrBlocks:
        - 192.168.0.0/16
    services:
      cidrBlocks:
        - 10.96.0.0/12
  controlPlaneConfiguration:
    count: 3
    endpoint:
      host: "10.0.0.100"  # Control Plane VIP
    machineGroupRef:
      kind: VSphereMachineConfig
      name: cp-machines
  datacenterRef:
    kind: VSphereDatacenterConfig
    name: vsphere-dc
  kubernetesVersion: "1.29"
  workerNodeGroupConfigurations:
    - count: 3
      machineGroupRef:
        kind: VSphereMachineConfig
        name: worker-machines
      name: md-0
---
apiVersion: anywhere.eks.amazonaws.com/v1alpha1
kind: VSphereDatacenterConfig
metadata:
  name: vsphere-dc
spec:
  datacenter: "Datacenter"
  server: "vcenter.example.com"
  network: "/Datacenter/network/VM Network"
  thumbprint: "AB:CD:EF:..."
  insecure: false
---
apiVersion: anywhere.eks.amazonaws.com/v1alpha1
kind: VSphereMachineConfig
metadata:
  name: cp-machines
spec:
  diskGiB: 100
  folder: "/Datacenter/vm/EKS"
  memoryMiB: 16384
  numCPUs: 4
  osFamily: ubuntu
  resourcePool: "/Datacenter/host/Cluster/Resources"
  template: "/Datacenter/vm/Templates/ubuntu-2204-kube-v1.29"
  users:
    - name: capv
      sshAuthorizedKeys:
        - "ssh-rsa AAAA..."
```

### Bare Metal 환경 (Tinkerbell)

```
□ 서버 하드웨어 정보 수집
  ├── BMC IP (IPMI/iDRAC/iLO)
  ├── BMC 사용자명/비밀번호
  └── MAC 주소

□ 네트워크 부팅 설정
  ├── DHCP 서버
  └── PXE 부팅 가능 확인

□ Tinkerbell 인프라 준비
```

**Bare Metal 하드웨어 정보 파일:**

```yaml
# hardware.csv
hostname,mac,ip,gateway,netmask,nameservers,bmc_ip,bmc_username,bmc_password
cp-1,00:50:56:xx:xx:x1,10.0.0.11,10.0.0.1,255.255.255.0,8.8.8.8,10.0.1.11,admin,password
cp-2,00:50:56:xx:xx:x2,10.0.0.12,10.0.0.1,255.255.255.0,8.8.8.8,10.0.1.12,admin,password
cp-3,00:50:56:xx:xx:x3,10.0.0.13,10.0.0.1,255.255.255.0,8.8.8.8,10.0.1.13,admin,password
worker-1,00:50:56:xx:xx:w1,10.0.0.21,10.0.0.1,255.255.255.0,8.8.8.8,10.0.1.21,admin,password
```

---

## Phase 3: 클러스터 생성

### 설정 파일 생성

```bash
□ 클러스터 설정 생성 (템플릿)
$ eksctl anywhere generate clusterconfig my-eks-anywhere \
  --provider vsphere > cluster-config.yaml

□ 설정 파일 편집
$ vi cluster-config.yaml
# 위의 예시 참고하여 환경에 맞게 수정
```

### 클러스터 생성

```bash
□ 설정 검증
$ eksctl anywhere create cluster -f cluster-config.yaml --dry-run

□ 클러스터 생성 실행
$ eksctl anywhere create cluster -f cluster-config.yaml

# 진행 상황 모니터링 (다른 터미널에서)
$ watch kubectl get machines -A

□ 클러스터 생성 완료 대기 (~30분)
```

### kubeconfig 설정

```bash
□ kubeconfig 파일 확인
$ ls -la my-eks-anywhere/
# my-eks-anywhere-eks-a-cluster.kubeconfig 파일 생성됨

□ KUBECONFIG 환경변수 설정
$ export KUBECONFIG=${PWD}/my-eks-anywhere/my-eks-anywhere-eks-a-cluster.kubeconfig

□ 클러스터 접근 확인
$ kubectl cluster-info
$ kubectl get nodes
$ kubectl get pods -A
```

---

## Phase 4: 필수 컴포넌트 확인

### Control Plane 컴포넌트

```bash
□ etcd 상태 확인
$ kubectl get pods -n kube-system -l component=etcd

□ API Server 상태 확인
$ kubectl get pods -n kube-system -l component=kube-apiserver

□ Controller Manager 상태 확인
$ kubectl get pods -n kube-system -l component=kube-controller-manager

□ Scheduler 상태 확인
$ kubectl get pods -n kube-system -l component=kube-scheduler
```

### 네트워킹 (Cilium)

```bash
□ Cilium 상태 확인
$ kubectl get pods -n kube-system -l app.kubernetes.io/part-of=cilium
$ cilium status

□ Cilium 연결 테스트
$ cilium connectivity test
```

### GitOps (Flux)

EKS Anywhere는 Flux가 기본 내장되어 있습니다.

```bash
□ Flux 컴포넌트 확인
$ kubectl get pods -n flux-system

□ GitOps 소스 확인
$ kubectl get gitrepositories -A
$ kubectl get kustomizations -A
```

---

## Phase 5: 검증

### 기본 기능 테스트

```bash
□ 테스트 애플리케이션 배포
$ kubectl create deployment nginx --image=nginx
$ kubectl expose deployment nginx --port=80 --type=NodePort

□ 서비스 접근 테스트
$ kubectl get svc nginx
$ curl http://<node-ip>:<node-port>

□ Pod 스케일링 테스트
$ kubectl scale deployment nginx --replicas=3
$ kubectl get pods -l app=nginx -o wide

□ 테스트 리소스 정리
$ kubectl delete deployment nginx
$ kubectl delete svc nginx
```

### Control Plane 고가용성 테스트

```bash
□ Control Plane 노드 확인
$ kubectl get nodes -l node-role.kubernetes.io/control-plane

□ 하나의 CP 노드 종료 후 클러스터 동작 확인
# (프로덕션 전 테스트 환경에서만!)
$ kubectl get nodes  # 2개의 CP가 Ready여야 함
$ kubectl get pods   # Pod 조회 가능해야 함

□ CP 노드 복구 후 재조인 확인
```

### etcd 백업 테스트

```bash
□ etcd 백업 생성
$ ETCDCTL_API=3 etcdctl snapshot save /tmp/etcd-backup.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

□ 백업 검증
$ ETCDCTL_API=3 etcdctl snapshot status /tmp/etcd-backup.db
```

---

## Phase 6: 모니터링 설정

### Prometheus + Grafana

```bash
□ Helm 설치 (아직 없다면)
$ curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

□ Prometheus Operator 설치
$ helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
$ helm repo update

□ kube-prometheus-stack 설치
$ kubectl create namespace monitoring
$ helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set grafana.adminPassword=admin123

□ 설치 확인
$ kubectl get pods -n monitoring

□ Grafana 접근
$ kubectl port-forward svc/prometheus-grafana 3000:80 -n monitoring
# 브라우저: http://localhost:3000
# 로그인: admin / admin123
```

### 로그 수집 (Loki + Promtail)

```bash
□ Loki 설치
$ helm repo add grafana https://grafana.github.io/helm-charts
$ helm install loki grafana/loki-stack \
  --namespace monitoring \
  --set promtail.enabled=true

□ Grafana에서 Loki 데이터 소스 추가
# URL: http://loki:3100
```

---

## Phase 7: 백업 설정

### Velero 설치

```bash
□ MinIO 설치 (온프레미스 S3 호환 스토리지)
$ kubectl create namespace velero
$ helm repo add minio https://charts.min.io/
$ helm install minio minio/minio \
  --namespace velero \
  --set rootUser=admin \
  --set rootPassword=minio123 \
  --set persistence.size=50Gi

□ Velero 설치
$ velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.7.0 \
  --bucket velero \
  --backup-location-config region=minio,s3ForcePathStyle=true,s3Url=http://minio.velero:9000 \
  --use-volume-snapshots=false \
  --secret-file ./credentials-velero
```

**credentials-velero 파일:**

```
[default]
aws_access_key_id = admin
aws_secret_access_key = minio123
```

### 정기 백업 스케줄

```bash
□ 백업 스케줄 생성
$ velero schedule create daily-backup \
  --schedule="0 2 * * *" \
  --ttl 168h

□ 백업 테스트
$ velero backup create test-backup --include-namespaces default
$ velero backup describe test-backup
```

### etcd 정기 백업 (CronJob)

```yaml
# etcd-backup-cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: etcd-backup
  namespace: kube-system
spec:
  schedule: "0 */6 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          hostNetwork: true
          tolerations:
          - key: node-role.kubernetes.io/control-plane
            effect: NoSchedule
          nodeSelector:
            node-role.kubernetes.io/control-plane: ""
          containers:
          - name: etcd-backup
            image: bitnami/etcd:3.5
            command:
            - /bin/sh
            - -c
            - |
              etcdctl snapshot save /backup/etcd-$(date +%Y%m%d-%H%M).db \
                --endpoints=https://127.0.0.1:2379 \
                --cacert=/etc/kubernetes/pki/etcd/ca.crt \
                --cert=/etc/kubernetes/pki/etcd/server.crt \
                --key=/etc/kubernetes/pki/etcd/server.key
              find /backup -name "*.db" -mtime +7 -delete
            volumeMounts:
            - name: etcd-certs
              mountPath: /etc/kubernetes/pki/etcd
              readOnly: true
            - name: backup
              mountPath: /backup
          volumes:
          - name: etcd-certs
            hostPath:
              path: /etc/kubernetes/pki/etcd
          - name: backup
            hostPath:
              path: /var/lib/etcd-backup
          restartPolicy: OnFailure
```

---

## Phase 8: 클러스터 업그레이드 준비

### 업그레이드 절차 문서화

```
□ 업그레이드 전 체크리스트 작성
  ├── etcd 백업 완료
  ├── 애플리케이션 백업 (Velero)
  ├── Release Notes 검토
  └── 롤백 계획 수립

□ 업그레이드 명령 확인
$ eksctl anywhere upgrade cluster -f cluster-config.yaml

□ 업그레이드 테스트 (테스트 환경에서)
```

---

## 체크리스트 요약

```
Pre-requisites:
□ Admin Machine 준비 (Docker, eksctl-anywhere)
□ 인프라 요구사항 확인 (vSphere/Bare Metal)
□ 네트워크 IP 계획
□ 하드웨어 준비 (CP 3대, Worker 3대+)

Infrastructure Setup:
□ vSphere OVA 템플릿 업로드 (또는 Bare Metal BMC 설정)
□ 클러스터 설정 파일 작성

Cluster Creation:
□ 클러스터 생성 (eksctl anywhere create cluster)
□ kubeconfig 설정

Verification:
□ Control Plane 컴포넌트 확인
□ Cilium 상태 확인
□ 테스트 애플리케이션 배포

Monitoring:
□ Prometheus + Grafana 설치
□ 로그 수집 설정 (Loki)

Backup:
□ Velero 설치 및 설정
□ etcd 백업 자동화
□ 정기 백업 스케줄 설정

Operations:
□ 업그레이드 절차 문서화
□ 장애 대응 절차 문서화
```

---

## 트러블슈팅

### 클러스터 생성 실패

```bash
# 상세 로그 확인
$ eksctl anywhere create cluster -f cluster-config.yaml -v 9

# 일반적인 원인:
# 1. vCenter 연결 문제 (인증, 네트워크)
# 2. OVA 템플릿 문제
# 3. 리소스 부족 (CPU, RAM)
# 4. 네트워크 설정 오류
```

### Control Plane 노드 장애

```bash
# etcd 클러스터 상태 확인
$ kubectl exec -it etcd-<node> -n kube-system -- etcdctl member list

# 장애 노드 제거 (필요시)
$ kubectl exec -it etcd-<healthy-node> -n kube-system -- etcdctl member remove <member-id>

# 노드 재조인
$ eksctl anywhere upgrade cluster -f cluster-config.yaml
```

### 네트워크 문제

```bash
# Cilium 상태 확인
$ cilium status
$ cilium connectivity test

# Pod 간 통신 테스트
$ kubectl run test1 --image=busybox --restart=Never -- sleep 3600
$ kubectl run test2 --image=busybox --restart=Never -- sleep 3600
$ kubectl exec -it test1 -- ping <test2-ip>
```

---

> ⚠️ **주의**
>
> **중요**: EKS Anywhere는 Control Plane을 직접 관리하므로 etcd 백업이 매우 중요합니다. 반드시 정기 백업을 설정하고 복구 테스트를 수행하세요.

> ✅ **완료**
>
> 체크리스트를 완료했다면 [Best Practices](best-practices.md)를 참고하여 프로덕션 환경을 더 안정적으로 운영하세요.
