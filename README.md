# In-house LLM Inference on Kubernetes — Learning Repo

A hands-on breakdown of the blog [In-house LLM Inference on Kubernetes: A Production Runbook](https://gd03.me/writings/inference-infra) by GD03Champ (repo: [gd03champ/inference-infra](https://github.com/gd03champ/inference-infra)).

The original post is a dense, single-page runbook. This repo turns it into:

1. A clean, **readable** version of the article — [`docs/blog.md`](docs/blog.md)
2. **Architecture diagrams** — full stack, request flow, Pi + Bifrost CLI consumption — [`docs/architecture.md`](docs/architecture.md)
3. **Small concept modules** you can learn one at a time — [`docs/modules/`](docs/modules/)
4. **Hands-on labs** you can actually run (locally, no AWS bill) — [`docs/labs/`](docs/labs/)
5. **Alternative-stack deep dives** to learn by swapping pieces — [`docs/alternatives/`](docs/alternatives/)
   - Replace Bifrost with **Envoy AI Gateway**
   - Explore the **Bifrost** UI/gateway itself
   - Add **HAMi** GPU sharing (fractional GPUs — the thing the blog says is impossible with the vanilla device plugin)

---

## The stack in one picture

Production (Track B), learning tracks (A/C/D), request flow, autoscaling, and **Pi / Bifrost CLI** wiring — all in [`docs/architecture.md`](docs/architecture.md).

```mermaid
flowchart TB
    subgraph clients["CLIENTS"]
        pi["Pi harness"]
        bfcli["Bifrost CLI"]
        apps["Claude Code · Open WebUI · curl"]
    end

    gw["Istio Gateway (TLS)"]
    bifrost["Bifrost · routing · fallback · cost logs"]
    epp["llm-d EPP · prefix routing · KV reuse"]
    vllm["vLLM · Qwen2.5-Coder-14B-AWQ"]
    gpu["GPU nodes · Karpenter JIT"]
    keda["KEDA · queue depth"]

    clients --> gw --> bifrost --> epp --> vllm --> gpu
    vllm -.-> keda
    keda -.-> gpu
```

---

## Suggested learning path

Work top to bottom. Each module is short; each lab is runnable.

| # | Read | Then do |
|---|------|---------|
| 0 | [`docs/blog.md`](docs/blog.md) — the whole story, readable | — |
| 0b | [`docs/homelab-production-stack.md`](docs/homelab-production-stack.md) — **your target: full stack on L4** | [`manifests/homelab/`](../manifests/homelab/) |
| 1 | [`modules/01-gpus-in-k8s.md`](docs/modules/01-gpus-in-k8s.md) | [`labs/lab-00-local-cluster.md`](docs/labs/lab-00-local-cluster.md) |
| 2 | [`modules/02-karpenter-gpu-nodes.md`](docs/modules/02-karpenter-gpu-nodes.md) | [`lab-07` Jarvis](docs/labs/lab-07-jarvislabs-gpu.md) · [`lab-08` Modal](docs/labs/lab-08-modal-serverless.md) · [`lab-01` vLLM](docs/labs/lab-01-vllm-single-gpu.md) |
| 3 | [`modules/03-ingress-tls.md`](docs/modules/03-ingress-tls.md) | — |
| 4 | [`modules/04-observability.md`](docs/modules/04-observability.md) | (Prometheus/Grafana in lab-01) |
| 5 | [`modules/05-serving-llm-d.md`](docs/modules/05-serving-llm-d.md) | [`labs/lab-01-vllm-single-gpu.md`](docs/labs/lab-01-vllm-single-gpu.md) |
| 6 | [`modules/06-scaling-keda.md`](docs/modules/06-scaling-keda.md) | [`labs/lab-05-keda-autoscale.md`](docs/labs/lab-05-keda-autoscale.md) |
| 7 | [`modules/07-economics.md`](docs/modules/07-economics.md) | — |
| 8 | [`alternatives/hami-gpu-sharing.md`](docs/alternatives/hami-gpu-sharing.md) | [`labs/lab-02-hami-gpu-sharing.md`](docs/labs/lab-02-hami-gpu-sharing.md) |
| 9 | [`alternatives/envoy-ai-gateway.md`](docs/alternatives/envoy-ai-gateway.md) | [`labs/lab-03-envoy-ai-gateway.md`](docs/labs/lab-03-envoy-ai-gateway.md) |
| 10 | [`alternatives/bifrost.md`](docs/alternatives/bifrost.md) | [`labs/lab-04-bifrost.md`](docs/labs/lab-04-bifrost.md) |
| 11 | [`docs/architecture.md`](docs/architecture.md) — full diagrams | [`labs/lab-09-consume-pi-bifrost-cli.md`](docs/labs/lab-09-consume-pi-bifrost-cli.md) |

---

## Three ways to run the labs

You do **not** need to burn money on an H200 to learn this. Four tracks:

- **Track A — Local (free).** `kind` or `minikube` on your MacBook. Learn Kubernetes, Gateway API, KEDA, Bifrost, Envoy AI Gateway, and (if you have any NVIDIA GPU) HAMi + a small vLLM model. This is where you should start. See [`labs/lab-00-local-cluster.md`](docs/labs/lab-00-local-cluster.md).
- **Track E — Homelab production (recommended goal).** **Same stack as the blog** (kind, llm-d, Bifrost, KEDA, Prometheus, DCGM, **k9s** UI) on **Jarvis L4 + Qwen2.5-Coder-14B-AWQ** — no AWS. See [`docs/homelab-production-stack.md`](docs/homelab-production-stack.md).
- **Track C — Jarvis Labs GPU (lowest ₹/hr warm).** Real vLLM on **L4 ~₹36/hr** (India IN2). SSH + Docker + memory tuning. See [`labs/lab-07-jarvislabs-gpu.md`](docs/labs/lab-07-jarvislabs-gpu.md).
- **Track D — Modal serverless (scale-to-zero).** vLLM on **L4 ~$0.80/hr (~₹66/hr)** but billed per second; **$30/mo free credit**. Best for bursty sessions. Skips K8s. See [`labs/lab-08-modal-serverless.md`](docs/labs/lab-08-modal-serverless.md).
- **Track B — AWS (reference).** Original blog on EKS + Karpenter. After Track E works on homelab.

> Model names in the blog (`glm-5.2`, `qwen-3.6`, `sonnet-5`) are the author's near-future naming. In the labs we substitute small, really-downloadable models (e.g. `Qwen/Qwen2.5-0.5B-Instruct`, `facebook/opt-125m`) so you can actually run them.

---

## Prerequisites checklist

- `kubectl`, `helm` (v3.12+), `git`, `curl`, [`k9s`](https://k9scli.io/) (optional cluster UI)
- Docker + **kind** (homelab cluster on Jarvis; Lab 00 on Mac)
- For real GPU labs: [Jarvis Labs](https://jarvislabs.ai) (L4 ~₹36/hr India) or [Modal](https://modal.com) (L4 scale-to-zero, $30/mo free credit), or AWS for Karpenter
- Optional but recommended: a [Hugging Face](https://huggingface.co/settings/tokens) token for gated models

See [`docs/labs/lab-00-local-cluster.md`](docs/labs/lab-00-local-cluster.md) to get a cluster up in ~5 minutes.
