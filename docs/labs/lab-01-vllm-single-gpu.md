# Lab 01 — Run vLLM (a real OpenAI-compatible model server)

**Goal:** serve a tiny model with vLLM and hit it with the OpenAI API — the atom of the whole stack.
**You'll learn:** `vllm serve` flags, the OpenAI-compatible endpoint, and the memory knobs from Module 5.

Two paths: **Docker (fastest, no cluster)** then **Kubernetes (closer to the blog)**.

---

## Path A — Docker on your laptop (CPU or GPU)

### A1. CPU-only (works on any machine, slow but real)

```bash
docker run --rm -p 8000:8000 \
  vllm/vllm-openai:latest \
  --model Qwen/Qwen2.5-0.5B-Instruct \
  --max-model-len 4096 \
  --max-num-seqs 8 \
  --device cpu
```

> A 0.5B model is chosen so it downloads fast and fits in RAM. First run pulls the model from Hugging Face.

### A2. With an NVIDIA GPU

```bash
docker run --rm --gpus all -p 8000:8000 \
  vllm/vllm-openai:latest \
  --model Qwen/Qwen2.5-0.5B-Instruct \
  --gpu-memory-utilization 0.85 \
  --max-model-len 8192 \
  --max-num-seqs 16
```

### A3. Call it (identical to calling OpenAI/Claude)

```bash
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen/Qwen2.5-0.5B-Instruct",
    "messages": [{"role":"user","content":"Say hi in 5 words."}]
  }'
```

You now have the *exact* API contract Bifrost/Envoy AI Gateway sit in front of.

---

## Path B — vLLM on Kubernetes (kind cluster from Lab 00)

### B1. Deployment + Service

```bash
kubectl create namespace serving
kubectl apply -n serving -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vllm-qwen
  labels: {app: vllm-qwen}
spec:
  replicas: 1
  selector: {matchLabels: {app: vllm-qwen}}
  template:
    metadata: {labels: {app: vllm-qwen}}
    spec:
      containers:
        - name: vllm
          image: vllm/vllm-openai:latest
          args:
            - "--model=Qwen/Qwen2.5-0.5B-Instruct"
            - "--max-model-len=4096"
            - "--max-num-seqs=8"
            - "--device=cpu"        # drop this + add GPU limits on a real GPU node
          ports: [{containerPort: 8000}]
          # On a GPU node, add:
          # resources: {limits: {nvidia.com/gpu: 1}}
          # and tolerations/nodeSelector from Lab 00
---
apiVersion: v1
kind: Service
metadata: {name: vllm-qwen}
spec:
  selector: {app: vllm-qwen}
  ports: [{port: 80, targetPort: 8000}]
EOF

kubectl -n serving rollout status deploy/vllm-qwen --timeout=600s
```

### B2. Test through the Service

```bash
kubectl -n serving port-forward svc/vllm-qwen 8000:80 &
curl http://localhost:8000/v1/models
```

The in-cluster URL `http://vllm-qwen.serving.svc.cluster.local` is the local equivalent of the blog's
`http://optimized-baseline-qwen36-epp.infra-qwen36.svc.cluster.local:80/v1`.

---

## Experiment: reproduce the OOM lesson

On a GPU, deliberately over-request and watch it crash, then fix it — the blog's core tuning loop:

1. Set `--max-num-seqs=256` (the vLLM default) with a model near your VRAM ceiling → likely `CUDA out of memory`.
2. Drop to `--max-num-seqs=16`, add `--gpu-memory-utilization=0.90` → it starts.
3. Raise `max-num-seqs` step by step until it OOMs, back off one. That's your ceiling.

This *is* the benchmark grind from [Module 5](../modules/05-serving-llm-d.md), just at laptop scale.

## Optional: wire vLLM metrics into Prometheus

vLLM exposes `/metrics` (Prometheus format). With kube-prometheus-stack installed, add a `ServiceMonitor` — **remember the `release: kube-prometheus-stack` label** ([Module 4](../modules/04-observability.md)) or you'll get no data. Those metrics feed KEDA in [Lab 05](lab-05-keda-autoscale.md).

**Next:** [Lab 05 — Autoscale with KEDA](lab-05-keda-autoscale.md) · or jump to [Lab 04 — Bifrost](lab-04-bifrost.md) to put a gateway in front.
