# 데이터 보안

> "Secret은 어떻게 안전하게 관리하나요?"

민감한 데이터(비밀번호, API 키, 인증서 등)를 안전하게 관리하는 것은 매우 중요합니다.

## Kubernetes Secret의 한계

```
┌─────────────────────────────────────────────────────────────────┐
│                 Kubernetes Secret 기본 동작                     │
│                                                                 │
│   ⚠️  기본 Secret은 base64 인코딩일 뿐, 암호화가 아님!        │
│                                                                 │
│   # 쉽게 디코딩 가능                                           │
│   $ echo "cGFzc3dvcmQ=" | base64 -d                            │
│   password                                                      │
│                                                                 │
│   문제점:                                                       │
│   • etcd에 평문으로 저장 (기본)                                │
│   • RBAC 없으면 누구나 읽기 가능                               │
│   • Git에 실수로 커밋 가능                                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## Secret 암호화

### etcd 암호화 (EKS Anywhere)

```yaml
# encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: <base64-encoded-32-byte-key>
      - identity: {}
```

### EKS Hybrid Nodes

EKS에서는 AWS KMS로 etcd가 자동 암호화됩니다.

## 외부 시크릿 관리자

### External Secrets Operator

```yaml
# AWS Secrets Manager 연동
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secrets-manager
  namespace: production
spec:
  provider:
    aws:
      service: SecretsManager
      region: ap-northeast-2
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets-sa
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: database-credentials
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: SecretStore
  target:
    name: database-credentials
  data:
  - secretKey: username
    remoteRef:
      key: prod/database
      property: username
  - secretKey: password
    remoteRef:
      key: prod/database
      property: password
```

### HashiCorp Vault

```yaml
# Vault CSI Provider
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: vault-database
spec:
  provider: vault
  parameters:
    vaultAddress: "https://vault.example.com"
    roleName: "my-app"
    objects: |
      - objectName: "db-password"
        secretPath: "secret/data/database"
        secretKey: "password"
```

## 전송 중 암호화

### mTLS with Istio

```yaml
# 네임스페이스에 mTLS 강제
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT
```

### Cilium 암호화

```yaml
# Cilium Encryption (WireGuard)
# values.yaml
encryption:
  enabled: true
  type: wireguard
```

## 백업 암호화

```yaml
# Velero 암호화 백업
apiVersion: velero.io/v1
kind: BackupStorageLocation
metadata:
  name: default
spec:
  provider: aws
  objectStorage:
    bucket: my-backup-bucket
  config:
    region: ap-northeast-2
    s3ForcePathStyle: "true"
    serverSideEncryption: aws:kms
    kmsKeyId: alias/velero-backup-key
```

## 모범 사례

```
┌─────────────────────────────────────────────────────────────────┐
│                   데이터 보안 모범 사례                         │
│                                                                 │
│   시크릿 관리:                                                  │
│   ✅ 외부 시크릿 관리자 사용 (Vault, AWS SM)                   │
│   ✅ Secret을 Git에 절대 저장하지 않음                         │
│   ✅ 시크릿 로테이션 자동화                                    │
│   ✅ 접근 감사 로깅                                            │
│                                                                 │
│   암호화:                                                       │
│   ✅ etcd 암호화 활성화                                        │
│   ✅ 전송 중 암호화 (TLS/mTLS)                                 │
│   ✅ 백업 암호화                                               │
│                                                                 │
│   접근 제어:                                                    │
│   ✅ Secret에 대한 RBAC 엄격히 적용                            │
│   ✅ 필요한 Pod에만 마운트                                     │
│   ✅ 환경변수보다 파일 마운트 선호                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

[다음: 네트워크 →](../08-networking/README.md)
