# Lab 07 — Production vLLM on Jarvis Labs (cheap GPU track)

**Goal:** run real GPU inference with production `vllm serve` tuning on [Jarvis Labs](https://jarvislabs.ai) — same skills as the blog, without AWS EKS cost.
**You'll learn:** GPU pick by VRAM, pause/resume billing, memory tuning, gateway wiring from your MacBook.
**Runs on:** Jarvis Labs GPU VM (India region **IN2**, Noida) + your MacBook as client.

> **Pricing:** Jarvis India on-demand, per-minute billing, pause anytime. INR derived from official rate **H100 = ₹217.89/hr = $2.69/hr** ([Jarvis blog](https://jarvislabs.ai/blog/h100-price-india)) → **~₹81 per $1**. Verify live rates at [jarvislabs.ai/pricing](https://jarvislabs.ai/pricing) (toggle India) or `jl gpus`.

---

## Which GPU to pick (exact specs + INR)

For **one user learning production inferencing**, use **24 GB** cards. Skip H100/H200 unless you're exploring 70B+.

| GPU | VRAM | System RAM | vCPU | Architecture | ₹/hr (on-demand) | $/hr | Pick for |
|-----|------|------------|------|--------------|------------------|------|----------|
| **NVIDIA A30** | **24 GB HBM2** | **112 GB** | **16** | Ampere (sm_80) | **~₹33** | $0.41 | Cheapest 24 GB; learning + 14B AWQ |
| **NVIDIA L4** ⭐ | **24 GB GDDR6** | **124 GB** | **32** | Ada Lovelace (sm_89) | **~₹36** | $0.44 | **Default** — faster inference, more RAM for HF cache |
| NVIDIA A100 40GB | 40 GB HBM2 | 112 GB | 16 | Ampere | ~₹72 | $0.89 | 32B AWQ at 32k context, higher `max-num-seqs` |
| NVIDIA A100 80GB | 80 GB HBM2 | 112 GB | 16 | Ampere | ~₹121 | $1.49 | 70B quant — skip for learning |
| NVIDIA H100 SXM | 80 GB HBM3 | 200 GB | 16 | Hopper | **₹218** | $2.69 | Skip for learning |

**L4 extra specs:** 7,424 CUDA cores · 232 Tensor cores · 300 GB/s memory bandwidth · 72 W TDP · native FP8.

**Recommendation:** **L4, 1× GPU, region IN2** (~₹36/hr). Budget tighter? **A30** (~₹33/hr).

**Blog mapping:** AWS `g6.xlarge` (L4, 24 GB) ≈ Jarvis **L4**. AWS `g5` (A10G, 24 GB) ≈ Jarvis **L4/A30**.

### Rough session costs (pause when done)

| Session | GPU | Time | ~Cost (INR) |
|---------|-----|------|-------------|
| First vLLM boot + API test | L4 | 1 hr | ~₹36 |
| OOM tuning loop (Module 5) | L4 | 2 hr | ~₹72 |
| Wire Bifrost from Mac | L4 | 30 min | ~₹18 |
| Weekly learning | L4 | 3 hr | ~₹108 |
| Left on overnight (don't) | L4 | 8 hr | ~₹288 |

Spot instances (up to ~56% off) exist — OK for retry-tolerant learning, bad mid-debug.

---

## Exact models (quantized Hugging Face repos)

Use **pre-quantized AWQ repos** — do not pass `--quantization awq` on the base (non-AWQ) weights.

| Tier | Hugging Face model ID | Quant | Weight size | Fits on |
|------|----------------------|-------|-------------|---------|
| **Daily driver** ⭐ | `Qwen/Qwen2.5-Coder-14B-Instruct-AWQ` | **AWQ 4-bit** | ~9 GB | L4 / A30 with KV headroom |
| **Best quality (tight)** | `Qwen/Qwen2.5-Coder-32B-Instruct-AWQ` | **AWQ 4-bit** | ~18 GB | L4 / A30 at 16k ctx, `max-num-seqs=2` |
| **Fallback / first boot** | `Qwen/Qwen2.5-Coder-7B-Instruct-AWQ` | **AWQ 4-bit** | ~5 GB | anything; fast download |

License: Apache-2.0. No HF token required for these repos.

---

## Production vLLM commands (copy-paste)

**L4 (Ada)** — use `awq_marlin` for best throughput on sm_89:

### Primary: Coder 14B AWQ

```bash
docker run -d --name vllm --gpus all -p 8000:8000 \
  -v hf-cache:/root/.cache/huggingface \
  vllm/vllm-openai:latest \
  --model Qwen/Qwen2.5-Coder-14B-Instruct-AWQ \
  --quantization awq_marlin \
  --served-model-name Qwen2.5-Coder-14B-AWQ \
  --gpu-memory-utilization 0.90 \
  --max-model-len 32768 \
  --max-num-seqs 4 \
  --enable-auto-tool-choice \
  --tool-call-parser hermes
```

### Stretch: Coder 32B AWQ (24 GB wall lesson)

```bash
docker run -d --name vllm --gpus all -p 8000:8000 \
  -v hf-cache:/root/.cache/huggingface \
  vllm/vllm-openai:latest \
  --model Qwen/Qwen2.5-Coder-32B-Instruct-AWQ \
  --quantization awq_marlin \
  --served-model-name Qwen2.5-Coder-32B-AWQ \
  --gpu-memory-utilization 0.95 \
  --max-model-len 16384 \
  --max-num-seqs 2 \
  --enforce-eager \
  --enable-auto-tool-choice \
  --tool-call-parser hermes
```

**A30 (Ampere)** — if `awq_marlin` errors, use `--quantization awq` instead.

### Smoke test

```bash
curl http://localhost:8000/v1/models

curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen2.5-Coder-14B-AWQ",
    "messages": [{"role":"user","content":"Write a Go hello-world HTTP handler."}]
  }'
```

From MacBook, replace `localhost` with `<JARVIS_IP>` and expose port 8000 in the Jarvis UI.

---

## Setup steps

1. Sign up at [jarvislabs.ai](https://jarvislabs.ai) → region **IN2** (Noida).
2. **Create** → **NVIDIA L4** → **1× GPU** → template with Docker/CUDA (or SSH VM).
3. SSH in → `nvidia-smi` → run Docker command above.
4. Point Claude Code / Continue / Open WebUI at `http://<JARVIS_IP>:8000/v1`.
5. **Pause** instance when done.

### Gateway on Mac (production pattern)

See [Lab 04 — Bifrost](lab-04-bifrost.md): register `http://<JARVIS_IP>:8000` as provider; model name `Qwen2.5-Coder-14B-AWQ`.

---

## OOM tuning loop (Module 5)

On L4 with `Qwen/Qwen2.5-Coder-14B-Instruct-AWQ`:

1. `--max-num-seqs=256` → expect CUDA OOM.
2. `--max-num-seqs=4`, `--gpu-memory-utilization=0.90` → should start.
3. Raise `max-num-seqs` until OOM → back off one.

Same grind as [`docs/blog.md`](../blog.md) §5.

---

## Cost control

- [ ] **Pause** when not inferencing (per-minute billing).
- [ ] Persistent volume for HF cache only if reusing (~$0.10/GB/mo ≈ **₹8/GB/mo**).
- [ ] MacBook: free K8s/KEDA/Bifrost labs; Jarvis: GPU hours only.
- [ ] Calendar reminder — 8 hr overnight ≈ **₹288** on L4.

---

## Map to the blog

| Jarvis Labs | Blog (AWS EKS) |
|-------------|----------------|
| L4 / A30 (24 GB) | g5/g6 L4/A10G |
| Pause VM | Karpenter `consolidateAfter: 1m` |
| 1 vLLM container | llm-d + vLLM Deployment |
| Bifrost on Mac | Bifrost in cluster |
| `max-num-seqs=4` | `max-num-seqs=32` |

**Also consider:** [Lab 08 — Modal serverless](lab-08-modal-serverless.md) for scale-to-zero (different production skills).

**Next:** [Lab 08 — Modal](lab-08-modal-serverless.md) · [Lab 05 — KEDA on Mac](lab-05-keda-autoscale.md) · [Module 5](../modules/05-serving-llm-d.md)
