# Module 1 — GPUs in Kubernetes (the physics)

**Goal:** understand why a GPU is not "just another resource" like CPU/RAM. Everything else in the stack is shaped by this.

## The one rule that decides everything

By default, **a GPU is indivisible.** When a pod requests `nvidia.com/gpu: 1`, it takes the *entire* physical card — all of its VRAM, all of its compute. No other pod can share it. The `nvidia-device-plugin` enforces this.

```yaml
resources:
  limits:
    nvidia.com/gpu: 1   # = one whole physical GPU, e.g. all 24 GB of an L4
```

There is no `nvidia.com/gpu: 0.5`. If you try to over-schedule, pods would share the same memory pool and crash.

> This is the exact constraint **HAMi** removes (fractional GPUs). See [Module 8 / alternatives/hami-gpu-sharing.md](../alternatives/hami-gpu-sharing.md). Learning the default rule first is what makes HAMi make sense.

## VRAM is the budget

A model must fit in VRAM: **weights + KV cache + activation headroom.**

| Card | VRAM | Rough model ceiling (FP16) |
|------|------|----------------------------|
| L4 (g6) | 24 GB | ~7B–8B |
| A10G (g5) | 24 GB | ~7B–8B |
| L40S (g6e) | 48 GB | ~27B (quantized) |
| H100 | 80 GB | ~70B (quantized) |
| H200 | 141 GB | part of a 400B+ (needs 8×) |

Quantization (FP8, INT4) roughly halves/quarters the weight size, which is why the blog runs `Qwen3.6-27B-**FP8**` on a single 48 GB L40S.

## When one card isn't enough

Two ways to split a model:

- **Tensor Parallelism (TP)** — split each layer's math across GPUs **inside one node** (fast NVLink). `--tensor-parallel-size=8` = spread across 8 GPUs on one machine. Used for the GLM-5.2 8×H200 plan.
- **Pipeline Parallelism (PP)** — split *layers* across **separate nodes** (via Ray in vLLM). Slower interconnect; used only for the truly giant 500B+ models.

Rule of thumb: **TP within a node, PP across nodes, and avoid PP if you can.**

## The device plugin (the step that trips people)

Even on a node with drivers installed, `nvidia.com/gpu` will **not** appear as a schedulable resource until the NVIDIA device plugin DaemonSet is running. And if your GPU nodes are tainted (they should be), the plugin must **tolerate that taint** or it never lands.

```bash
kubectl get nodes -o custom-columns=NAME:.metadata.name,GPU:.status.allocatable.nvidia\\.com/gpu
```

If the `GPU` column is `<none>` on a GPU node, the device plugin isn't running there.

## Check your understanding

1. Why can't two pods each get half of one L4? → *Indivisible; shared memory pool would crash.*
2. Your 70B FP16 model needs ~140 GB. One node has 4×48 GB L40S. TP or PP? → *TP=4 within the node (192 GB total > 140 GB).*
3. `nvidia.com/gpu` shows `<none>` on a fresh GPU node. First thing to check? → *Device plugin DaemonSet + its taint tolerations.*

**Next:** [Module 2 — Karpenter & GPU nodes](02-karpenter-gpu-nodes.md) · **Lab:** [lab-00 local cluster](../labs/lab-00-local-cluster.md)
