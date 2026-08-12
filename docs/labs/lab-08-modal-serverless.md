# Lab 08 — Production vLLM on Modal (serverless GPU track)

**Goal:** deploy vLLM as a **scale-to-zero** OpenAI-compatible API on [Modal](https://modal.com) — learn serverless inference economics alongside the blog's Karpenter story.
**Runs on:** Modal cloud (USD billing) + your MacBook as client. No SSH VM to manage.

> **Can you use Modal here?** **Yes** — for vLLM serving, memory tuning, OpenAI API, Bifrost wiring, and cold-start economics. **No** — for K8s-specific pieces (Karpenter, KEDA in-cluster, device plugin, taints). Use [Lab 07](lab-07-jarvislabs-gpu.md) or Track A for those.

---

## Modal vs Jarvis Labs (pick your track)

| | **Modal (this lab)** | **Jarvis Labs (Lab 07)** |
|--|----------------------|--------------------------|
| Model | Serverless — scale to **zero** | VM — you **pause** manually |
| Billing | Per **second** of GPU time | Per **minute** while VM runs |
| L4-class 24 GB | ~**$0.80/hr** (~**₹66/hr** @ ₹83/$) | ~**$0.44/hr** (~**₹36/hr**) |
| Cold start | **30 s – 3 min** (model load) | **~0 s** if VM left running |
| Best for | Bursty coding sessions, pay only while generating | Long tuning sessions, SSH, Docker debugging |
| K8s blog skills | ❌ | Partial (Docker vLLM ✅, K8s ❌) |
| Free tier | **$30/month** compute credit (Starter) | No free GPU |

**Rule of thumb:**

- **Learning vLLM flags + short sessions** → Modal (scale-to-zero wins).
- **2+ hour OOM benchmark grind, SSH, `nvidia-smi`** → Jarvis L4 (~₹36/hr, cheaper per warm hour).
- **Full blog stack (KEDA, Karpenter, llm-d on K8s)** → Mac kind (Track A) + AWS (Track B).

---

## GPU + model to use on Modal

| Modal GPU | VRAM | $/hr | ~₹/hr | Model |
|-----------|------|------|-------|-------|
| **L4** ⭐ | 24 GB | $0.80 | ~₹66 | `Qwen/Qwen2.5-Coder-14B-Instruct-AWQ` |
| **A10** | 24 GB | $1.10 | ~₹91 | Same 14B AWQ (if L4 unavailable) |
| **L40S** | 48 GB | $1.95 | ~₹162 | `Qwen/Qwen2.5-Coder-32B-Instruct-AWQ` at 32k ctx |

Exact quantized model (same as Lab 07):

```
Qwen/Qwen2.5-Coder-14B-Instruct-AWQ   # AWQ 4-bit, ~9 GB weights
```

---

## Deploy (minimal)

### 1. Install Modal CLI (MacBook)

```bash
pip install modal
modal setup   # browser auth
```

### 2. Save as `modal_vllm_coder.py`

```python
import json
import subprocess

import modal

vllm_image = (
    modal.Image.from_registry("nvidia/cuda:12.9.0-devel-ubuntu22.04", add_python="3.12")
    .entrypoint([])
    .uv_pip_install("vllm==0.21.0")
)

MODEL = "Qwen/Qwen2.5-Coder-14B-Instruct-AWQ"
SERVED_NAME = "Qwen2.5-Coder-14B-AWQ"
VLLM_PORT = 8000

hf_cache = modal.Volume.from_name("hf-cache-coder", create_if_missing=True)
vllm_cache = modal.Volume.from_name("vllm-cache-coder", create_if_missing=True)

app = modal.App("llm-lab-coder")

MINUTES = 60

@app.function(
    image=vllm_image,
    gpu="L4",
    timeout=30 * MINUTES,
    volumes={
        "/root/.cache/huggingface": hf_cache,
        "/root/.cache/vllm": vllm_cache,
    },
)
@modal.web_server(port=VLLM_PORT, startup_timeout=10 * MINUTES)
def serve():
    cmd = [
        "vllm", "serve", MODEL,
        "--quantization", "awq_marlin",
        "--served-model-name", SERVED_NAME,
        "--host", "0.0.0.0",
        "--port", str(VLLM_PORT),
        "--gpu-memory-utilization", "0.90",
        "--max-model-len", "32768",
        "--max-num-seqs", "4",
        "--enforce-eager",  # faster cold starts while learning
        "--enable-auto-tool-choice",
        "--tool-call-parser", "hermes",
    ]
    subprocess.Popen(" ".join(cmd), shell=True)
```

### 3. Deploy

```bash
modal deploy modal_vllm_coder.py
```

Modal prints a URL like `https://your-workspace--llm-lab-coder-serve.modal.run`.

First request may return **503** for 1–3 min while vLLM loads — normal cold start.

### 4. Call it

```bash
export MODAL_URL="https://your-workspace--llm-lab-coder-serve.modal.run"

curl "$MODAL_URL/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen2.5-Coder-14B-AWQ",
    "messages": [{"role":"user","content":"Explain vLLM max-num-seqs in one paragraph."}]
  }'
```

Or from Python:

```python
from openai import OpenAI

client = OpenAI(base_url=f"{MODAL_URL}/v1", api_key="unused")
print(client.chat.completions.create(
    model="Qwen2.5-Coder-14B-AWQ",
    messages=[{"role": "user", "content": "Hello"}],
).choices[0].message.content)
```

### 5. Wire Bifrost (Mac)

[Lab 04](lab-04-bifrost.md): provider base URL = `https://your-workspace--llm-lab-coder-serve.modal.run`.

---

## Scale-to-zero settings (production economics)

Add to the `@app.function` decorator when you understand the defaults:

```python
scaledown_window=15 * MINUTES,  # stay warm 15 min after last request, then $0
```

| Setting | Learning value | Blog equivalent |
|---------|----------------|-----------------|
| `scaledown_window` | Idle cost vs cold-start tradeoff | Karpenter consolidation |
| `gpu="L4"` | VRAM budget | g6.xlarge |
| `enforce-eager` | Faster boot, slower tokens | `--max-num-seqs` tuning |
| HF + vLLM Volumes | Don't re-download weights | Persistent disk on VM |

**Modal Starter:** $30 free credit/month ≈ **~37 hr of L4** if fully utilized — enough for this curriculum.

---

## What you learn vs what you skip

| Skill | Modal ✅ | Jarvis ✅ | kind K8s ✅ |
|-------|---------|-----------|-------------|
| `vllm serve` memory flags | ✅ | ✅ | ✅ (CPU) |
| OpenAI-compatible API | ✅ | ✅ | ✅ |
| Bifrost gateway | ✅ | ✅ | ✅ |
| OOM / `max-num-seqs` grind | ✅ | ✅ easier (SSH) | ✅ tiny model |
| Cold start / scale-to-zero | ✅ **core lesson** | manual pause | N/A |
| KEDA autoscale | ❌ Modal autoscales | ❌ | ✅ Lab 05 |
| Karpenter JIT nodes | ❌ | pause ≈ manual | simulated |
| llm-d / EPP routing | ❌ | optional | later |

---

## 32B on Modal (optional)

Change `gpu="L40S"` and model to `Qwen/Qwen2.5-Coder-32B-Instruct-AWQ`, `--max-model-len 16384`, `--max-num-seqs 2`. ~$1.95/hr (~₹162/hr) while warm.

---

## Cleanup

```bash
modal app stop llm-lab-coder
# or delete: modal app delete llm-lab-coder
```

Remove Bifrost provider when app is stopped (URL goes dead).

**Next:** [Lab 07 — Jarvis](lab-07-jarvislabs-gpu.md) · [Lab 05 — KEDA](lab-05-keda-autoscale.md) · [Module 7 — economics](../modules/07-economics.md)
