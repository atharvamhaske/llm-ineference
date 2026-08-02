# Lab 05 — Autoscale vLLM with KEDA (local, free)

**Goal:** scale a Deployment on a real serving metric, exactly like the blog — minus the cloud bill.
**You'll learn:** the `ScaledObject`, Prometheus-driven scaling, and the KEDA→pending-pod handoff.

Prereqs: Lab 00 cluster, Lab 01 vLLM running in `serving`, and kube-prometheus-stack.

## 1. Install monitoring + KEDA

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add kedacore https://kedacore.github.io/charts
helm repo update

helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace --wait

helm install keda kedacore/keda -n keda --create-namespace --wait
```

## 2. Scrape vLLM metrics

vLLM serves `/metrics`. Point Prometheus at it (note the mandatory label):

```bash
kubectl apply -n serving -f - <<'EOF'
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: vllm-qwen
  labels:
    release: kube-prometheus-stack   # REQUIRED or Prometheus ignores it (Module 4)
spec:
  selector: {matchLabels: {app: vllm-qwen}}
  endpoints:
    - {port: http, path: /metrics, interval: 10s}
EOF
```

> If your Service port is unnamed, name it `http` (add `name: http` under the Service port) so the ServiceMonitor can find it.

Confirm data: `kubectl -n monitoring port-forward svc/kube-prometheus-stack-prometheus 9090` then query
`vllm:num_requests_waiting` (or `vllm:num_requests_running`) in the Prometheus UI.

## 3. The ScaledObject (the blog's pattern, local scale)

The blog scales on `llm_d_epp_average_queue_size`. Without llm-d's EPP, the equivalent vLLM signal is **waiting requests**:

```bash
kubectl apply -n serving -f - <<'EOF'
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata: {name: vllm-qwen-scaler}
spec:
  scaleTargetRef: {name: vllm-qwen}
  minReplicaCount: 1
  maxReplicaCount: 4
  pollingInterval: 15
  advanced:
    horizontalPodAutoscalerConfig:
      behavior:
        scaleUp:
          stabilizationWindowSeconds: 30
          policies: [{type: Pods, value: 1, periodSeconds: 30}]
        scaleDown:
          stabilizationWindowSeconds: 120
          policies: [{type: Pods, value: 1, periodSeconds: 60}]
  triggers:
    - type: prometheus
      metricType: Value
      metadata:
        serverAddress: http://kube-prometheus-stack-prometheus.monitoring.svc:9090
        query: sum(vllm:num_requests_waiting)
        threshold: "3"
EOF
```

> Windows are shortened vs. the blog (they wait 5–10 min because real GPU pods warm slowly; your local 0.5B model starts in seconds).

## 4. Drive load and watch it scale

In one terminal, watch:

```bash
kubectl -n serving get hpa,scaledobject,pods -w
```

In another, hammer it:

```bash
for i in $(seq 1 200); do
  curl -s http://localhost:8000/v1/chat/completions \
    -H 'Content-Type: application/json' \
    -d '{"model":"Qwen/Qwen2.5-0.5B-Instruct","messages":[{"role":"user","content":"write a 300 word story"}]}' \
    > /dev/null &
done
```

You should see: waiting-requests climb → KEDA raises replicas → new pods appear → queue drains → after the cool-down, replicas fall back to 1.

## Map it to the blog

| Local | Blog |
|-------|------|
| `vllm:num_requests_waiting` | `llm_d_epp_average_queue_size` |
| new replica schedules instantly | new replica → **Pending** → Karpenter makes a GPU node |
| replicas drop to 1 | pods drain → node consolidated away → bill stops |

The only piece you're *not* exercising locally is **Karpenter** (node provisioning). Everything on the pod side is identical — and that's the half you can practice for free.

## Cleanup

```bash
kubectl delete scaledobject vllm-qwen-scaler -n serving
```

**Next:** [Lab 02 — HAMi GPU sharing](lab-02-hami-gpu-sharing.md) · [Lab 03 — Envoy AI Gateway](lab-03-envoy-ai-gateway.md)
