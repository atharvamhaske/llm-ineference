# Runbooks — production inference

Step-by-step execution guides (copy-paste commands, verification gates, troubleshooting).

| Runbook | Description |
|---------|-------------|
| [**single-l4-production-runbook.md**](single-l4-production-runbook.md) | **Start here.** 1× Jarvis L4, kind, k9s, vLLM, Bifrost, KEDA, Pi — full production stack without Karpenter |
| [multi-gpu-karpenter-stack.md](../multi-gpu-karpenter-stack.md) | After GPU #2: prefill + decode pools, Karpenter on EKS |

**Concept docs:** [homelab-production-stack.md](../homelab-production-stack.md) (overview) · [architecture.md](../architecture.md) (diagrams)

**Manifests:** [`manifests/homelab/`](../../manifests/homelab/)
