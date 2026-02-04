# GenAI 워크로드

> "온프레미스에서 AI 모델을 돌릴 수 있나요?"

GenAI 붐과 함께 온프레미스 GPU 인프라에 대한 관심이 높아지고 있습니다.

## GPU on Kubernetes

```
┌─────────────────────────────────────────────────────────────────┐
│                 GPU 워크로드 아키텍처                           │
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  GPU Node Pool                                           │  │
│   │                                                          │  │
│   │  ┌───────────────┐  ┌───────────────┐                   │  │
│   │  │   GPU Node 1  │  │   GPU Node 2  │                   │  │
│   │  │   NVIDIA A100 │  │   NVIDIA A100 │                   │  │
│   │  │               │  │               │                   │  │
│   │  │  [LLM Pod]    │  │ [Training Pod]│                   │  │
│   │  │  [Inference]  │  │ [Fine-tuning] │                   │  │
│   │  └───────────────┘  └───────────────┘                   │  │
│   │                                                          │  │
│   │  NVIDIA Device Plugin + GPU Operator                    │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## GPU 스케줄링

```yaml
# GPU 요청 Pod
apiVersion: v1
kind: Pod
metadata:
  name: llm-inference
spec:
  containers:
  - name: llm
    image: my-llm:latest
    resources:
      limits:
        nvidia.com/gpu: 1  # GPU 1개 요청
```

## 온프레미스 AI 인프라 이점

```
✅ 데이터 보안: 민감한 데이터가 외부로 나가지 않음
✅ 레이턴시: 실시간 추론에 유리
✅ 비용: 지속적 사용 시 클라우드 GPU보다 저렴
✅ GPU 가용성: 클라우드 GPU 부족 문제 회피
```

## 관련 도구

| 도구 | 용도 |
|------|------|
| **Kubeflow** | ML 파이프라인 |
| **Ray** | 분산 컴퓨팅 |
| **vLLM** | LLM 서빙 |
| **Triton** | NVIDIA 추론 서버 |

---

[다음: eBPF와 Service Mesh →](ebpf-service-mesh.md)
