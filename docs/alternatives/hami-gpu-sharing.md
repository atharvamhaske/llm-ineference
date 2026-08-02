# Alternative — HAMi: fractional GPU sharing

> **The blog says:** "When a pod gets 1 GPU, it fully takes over the GPU. Another pod can't share it. Fractional allocation is not possible."
> **HAMi says:** hold my beer.

## What HAMi is

**HAMi** (Heterogeneous AI Computing Virtualization Middleware, [project-hami.io](https://project-hami.io)) is a CNCF sandbox project that replaces the vanilla NVIDIA device plugin with a **virtualization layer**. It lets multiple pods **share one physical GPU** — each getting a slice of VRAM and compute, enforced in software (via `HAMi-core`, an intercepting CUDA library).

This directly attacks the blog's most expensive constraint: an idle-but-locked GPU.

## Why it matters for the blog's stack

| Problem in the blog | HAMi's answer |
|---------------------|---------------|
| A 7B model wastes most of a 24 GB L4 (all of it is locked to that one pod) | Pack several small models / replicas on one card |
| Dev/test pods each grab a whole GPU | Give each a 4 GB slice; run 5 on one card |
| Fine-grained cost control impossible below "1 whole GPU" | Bill/quota by `gpumem` and `gpucores` |

Where the blog scales **out** (more whole GPUs via Karpenter+KEDA), HAMi lets you also scale **in** (more workloads per GPU). They're complementary.

## The new resource knobs

HAMi extends Kubernetes with these container `limits`:

| Resource | Meaning |
|----------|---------|
| `nvidia.com/gpu` | number of **vGPUs** requested |
| `nvidia.com/gpumem` | VRAM per vGPU, in **MiB** |
| `nvidia.com/gpumem-percentage` | VRAM per vGPU as a **% of the card** (alternative to `gpumem`) |
| `nvidia.com/gpucores` | compute per vGPU, as **% of the card's SMs** |

Example — two pods sharing one card, 4 GB each:

```yaml
resources:
  limits:
    nvidia.com/gpu: 1        # 1 vGPU...
    nvidia.com/gpumem: 4000  # ...capped at 4000 MiB VRAM
    nvidia.com/gpucores: 30  # ...and 30% of compute (optional)
```

## How enforcement works (the honest version)

HAMi injects `HAMi-core`, which sets `CUDA_DEVICE_MEMORY_LIMIT` and `CUDA_DEVICE_SM_LIMIT` inside the container. The CUDA calls are intercepted so a pod can't exceed its VRAM slice, and compute is throttled toward its `gpucores` cap.

**Trade-offs vs. the blog's hard isolation:**

- It's **software isolation**, not hardware (unlike NVIDIA MIG, which physically partitions A100/H100). A misbehaving kernel is contained on memory but compute is time-sliced/best-effort by default.
- Great for **many small / bursty / dev** workloads. For a single latency-critical production model that needs the whole card (the blog's Qwen3.6 case), a **whole GPU is still the right call**.
- MIG (hardware partitioning) and HAMi can be combined; MIG gives stronger isolation on supported cards.

## When to reach for HAMi vs. the blog's approach

- **Use whole-GPU (blog):** one big model saturating a card; strict latency SLOs; you scale by adding cards.
- **Use HAMi:** many small models; dev/CI/test clusters; inference services that individually can't fill a card; cost control below "1 GPU" granularity.

## Learn it hands-on

Do [Lab 02 — HAMi GPU sharing](../labs/lab-02-hami-gpu-sharing.md). Requires an actual NVIDIA GPU (HAMi's memory limiting can't be simulated on CPU-only kind).

## Further reading
- Deploy with Helm: https://project-hami.io/docs/get-started/deploy-with-helm
- GPU partitioning tutorial: https://project-hami.io/tutorials/labs/gpu-partitioning
- Global config reference: https://project-hami.io/docs/userguide/configure
