# Module 7 — The economics (why do this at all?)

**Goal:** be able to defend in-house inference on a spreadsheet — and know the capabilities you literally cannot rent.

## The headline: concurrency-per-dollar

A hosted API is one line of code. Running your own is all the work in Modules 1–6. So it only makes sense past a certain scale. The metric that flips it: **how many concurrent users one owned node serves vs. what those requests would cost on a hosted API.**

One large node ≈ 30 concurrent users at 128k context. Once you serve a team, the math starts favoring ownership — *if* you're disciplined about idle cost.

## The three cost levers

### 1. Off-hours shutdown
GPU nodes shut down at night, fall back to Claude for stragglers. You don't pay for GPUs while everyone's asleep. This is exactly what Karpenter's aggressive consolidation (Module 2) automates.

### 2. Overflow spill
On queue overflow (say 30 deep), spill to the hosted provider instead of degrading everyone. **Size for the common case, not the peak.** The gateway (Bifrost / Envoy AI Gateway) does the fallback.

### 3. Throughput optimization
The 2.5× from the MTP benchmark grind (Module 5) is 2.5× more users per node → directly fewer dollars per user. Tuning *is* a cost lever.

## The spreadsheet trick

Price your self-hosted model **equal to the hosted equivalent** (e.g. `sonnet-5`) in the gateway. Now every request logs a cost *as if* it hit Claude. Real savings = (sum of those logged costs) − (actual cluster cost). The spreadsheet writes itself, and it's defensible to finance.

```
monthly savings = Σ(requests × hosted_price) − (GPU node-hours × node_price)
                                              − (overflow requests actually sent to hosted)
```

## What you cannot rent (the real reasons)

Cost gets you in the door. You stay for capabilities a hosted API can't sell:

- **Prompt caching you own.** Prefix-aware routing + KV-cache reuse (LMCache) computes the giant agent system prompt once. On a hosted API you get whatever caching they decide, priced how they decide.
- **Request tiering.** *You* decide what's an interactive agent turn (priority) vs. a background batch job (throughput). You can't tier requests you don't control.
- **Uncensored / domain / security models.** Run models that won't refuse legitimate security work, or ones fine-tuned on your domain. For a security team this is not a nice-to-have.
- **Your own models + data residency.** Train, quantize, serve — on your hardware, data never leaving your VPC. The platform is model-agnostic (same machinery served GLM-5.2 and Qwen3.6).

## The honest caveat

The blog's frontier plan (GLM-5.2 on 8×H200) **never ran** — `p5en.48xlarge` capacity wasn't available. HBM cards from hyperscalers are booked months ahead by big tech. **Reality check:** a single person often *can't* rent frontier hardware at all, funds or not. Plan around the GPUs you can actually get (the blog fell back to L40S + a 27B model).

## When in-house is NOT worth it

- Spiky, low, or unpredictable traffic → hosted wins (you'd pay for idle).
- You need the absolute frontier model and can't get the hardware.
- Tiny team, no k8s/GPU ops capacity → the operational cost dwarfs savings.

**Back to:** [README learning path](../../README.md) · **Or branch into alternatives:** [HAMi](../alternatives/hami-gpu-sharing.md) · [Envoy AI Gateway](../alternatives/envoy-ai-gateway.md) · [Bifrost](../alternatives/bifrost.md)
