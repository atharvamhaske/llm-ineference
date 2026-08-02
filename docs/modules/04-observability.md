# Module 4 — Observability (Prometheus + DCGM + Grafana)

**Goal:** know *what* to measure and *why* it's wired before serving a single token. You can't autoscale on a signal you don't collect.

## Three signal sources, one Prometheus

```
node/cluster metrics ─┐
GPU metrics (DCGM) ───┼──► Prometheus ──► Grafana dashboards
vLLM/EPP serving metrics ┘                └► KEDA reads these to scale
```

The blog installs observability **first**, on purpose: the autoscaler (Module 6) literally scales on a Prometheus query, so the metric pipeline must exist before scaling can.

## The pieces

- **kube-prometheus-stack** — Prometheus + Grafana + Alertmanager in one Helm chart. Stores data on gp3 PVCs.
- **DCGM exporter** — NVIDIA's GPU metrics (utilization, memory, temp, power). Runs **only on GPU nodes** and must **tolerate the GPU taint**.
- **vLLM / EPP metrics** — the serving engine exposes throughput, token latencies, KV-cache usage, and **queue depth** (the scaling signal). Collected via PodMonitor/ServiceMonitor.

## The gotcha that eats an afternoon

Every `ServiceMonitor` and `PodMonitor` must carry the label:

```yaml
metadata:
  labels:
    release: kube-prometheus-stack
```

Without it, this Prometheus's selector silently ignores the monitor. **No error, no data.** If a dashboard is empty, check this label first. The blog repeats it on every monitor for exactly this reason.

## Metrics that actually matter for LLM serving

| Metric | Why you care |
|--------|-------------|
| `nvidia_gpu_duty_cycle` / DCGM util | Is the expensive card actually busy? |
| GPU memory used | How close to OOM (the blog's recurring enemy) |
| vLLM tokens/sec | Throughput — the number the MTP grind optimized |
| KV-cache usage % | Cache pressure; near 100% = requests will queue |
| **running vs. waiting requests / queue size** | **The KEDA scaling signal** |
| Time-to-first-token, inter-token latency | User-perceived responsiveness |

## The signal that drives scaling

The blog builds a custom Grafana dashboard for **EPP** (the llm-d router) metrics — queue size, routing, per-pool state. The single most important one:

```promql
sum(llm_d_epp_average_queue_size{namespace="infra-qwen36"})
```

That exact query becomes the KEDA trigger in Module 6. Watch it climb under load, watch pods get added, watch it fall.

## Local-track

kube-prometheus-stack + a small vLLM model runs fine on `kind`/`minikube` (no GPU needed for the Prometheus/Grafana parts). DCGM needs a real NVIDIA GPU. Lab-01 wires vLLM metrics into Prometheus locally.

**Next:** [Module 5 — Serving with llm-d](05-serving-llm-d.md)
