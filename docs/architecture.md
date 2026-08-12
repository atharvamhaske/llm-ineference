# Architecture — full inference stack & client consumption

How the production stack fits together, the learning-track variants, and how to consume it with **Pi harness**, **Bifrost CLI**, or any OpenAI-compatible client.

---

## 1. Production stack (blog / Track B — AWS EKS)

The target architecture from [`blog.md`](blog.md): multi-user, autoscaling, one hardened gateway.

```mermaid
flowchart TB
    subgraph clients["CLIENTS"]
        pi["Pi harness (pi CLI)"]
        bfcli["Bifrost CLI"]
        cc["Claude Code / Continue / Open WebUI"]
        curl["curl / OpenAI SDK"]
    end

    subgraph edge["EDGE — INGRESS"]
        istio["Istio Gateway + cert-manager<br/>TLS · IP allowlist"]
    end

    subgraph gateway["AI GATEWAY"]
        bifrost["Bifrost<br/>routing · fallback · cost logs · virtual keys"]
    end

    subgraph serving["SERVING — per model (llm-d)"]
        epp["llm-d router / EPP (GAIE)<br/>prefix-aware routing · LMCache KV reuse"]
        vllm["vLLM model server<br/>Qwen2.5-Coder-14B-AWQ · Qwen3.6-27B-FP8"]
        epp --> vllm
    end

    subgraph compute["COMPUTE — GPU nodes"]
        karp["Karpenter<br/>JIT provision · consolidate idle"]
        prefill["prefill pool (spot)"]
        decode["decode pool (on-demand)"]
        karp --> prefill
        karp --> decode
    end

    subgraph scale["AUTOSCALE"]
        keda["KEDA<br/>EPP queue depth"]
    end

    subgraph obs["OBSERVABILITY"]
        prom["Prometheus + Grafana"]
        dcgm["DCGM exporter<br/>GPU metrics"]
    end

    subgraph fallback["OVERFLOW"]
        hosted["Hosted API<br/>Claude / OpenAI"]
    end

    clients --> istio --> bifrost
    bifrost --> epp
    bifrost -.->|queue overflow / off-hours| hosted
    vllm --> decode
    vllm -.-> prefill
    vllm -.->|Pending pod| karp
    epp -.->|queue metric| prom
    vllm -.->|/metrics| prom
    decode -.-> dcgm --> prom
    prom -.-> keda
    keda -.->|+replica| vllm
```

---

## 2b. Multi-GPU + Karpenter (Track F — production target)

Prefill (spot) + decode (on-demand), KEDA → Pending → Karpenter, llm-d EPP. Full doc: [`multi-gpu-karpenter-stack.md`](multi-gpu-karpenter-stack.md).

```mermaid
flowchart TB
    k9s["k9s"]
    bifrost["Bifrost"]
    epp["llm-d EPP"]
    prefill["vLLM prefill · spot pool"]
    decode["vLLM decode · on-demand pool"]
    keda["KEDA"]
    karp["Karpenter"]
    prefill_nodes["prefill L4 nodes"]
    decode_nodes["decode L4 nodes"]

    k9s -.-> decode_nodes
    bifrost --> epp
    epp --> prefill --> prefill_nodes
    epp --> decode --> decode_nodes
    epp -.-> keda -.-> karp
    karp --> prefill_nodes
    karp --> decode_nodes
```

**Karpenter requires EKS (AWS EC2).** Jarvis multi-VM uses the same taints/pools manually until you move to EKS.

## 2c. Single L4 homelab (Track E — starter)

[`homelab-production-stack.md`](homelab-production-stack.md) — kind + k9s, one GPU, decode only.

---

## 3. Learning stacks (Tracks A / C / D)

Same **OpenAI API contract** at every layer; fewer moving parts while you learn.

```mermaid
flowchart LR
    subgraph trackA["Track A — Mac (free)"]
        a_k8s["kind / minikube"]
        a_keda["KEDA + Prometheus"]
        a_bf["Bifrost Docker"]
        a_vllm["vLLM CPU<br/>Qwen2.5-0.5B"]
        a_k8s --> a_vllm
        a_keda -.-> a_vllm
        a_bf --> a_vllm
    end

    subgraph trackC["Track C — Jarvis Labs (~₹36/hr)"]
        c_gpu["L4 / A30 VM · IN2"]
        c_vllm["vLLM Docker<br/>Qwen2.5-Coder-14B-AWQ"]
        c_gpu --> c_vllm
    end

    subgraph trackD["Track D — Modal (scale-to-zero)"]
        d_fn["Modal @app.function gpu=L4"]
        d_vllm["vLLM subprocess<br/>Qwen2.5-Coder-14B-AWQ"]
        d_fn --> d_vllm
    end

    mac["MacBook<br/>Pi · Bifrost CLI · curl"] --> trackA
    mac --> trackC
    mac --> trackD
```

| Track | GPU | Model (quantized) | Gateway | K8s autoscale |
|-------|-----|-----------------|---------|---------------|
| A | CPU (kind) | `Qwen/Qwen2.5-0.5B-Instruct` | Bifrost local | KEDA ✅ |
| C | Jarvis L4 24 GB | `Qwen/Qwen2.5-Coder-14B-Instruct-AWQ` | Bifrost on Mac | manual pause |
| D | Modal L4 24 GB | `Qwen/Qwen2.5-Coder-14B-Instruct-AWQ` | Bifrost on Mac | Modal autoscale |
| B | EKS g5/g6 | blog models | Bifrost in cluster | KEDA + Karpenter ✅ |

---

## 3. Request flow (one chat completion)

```mermaid
sequenceDiagram
    participant C as Client<br/>(Pi / Bifrost CLI / curl)
    participant B as Bifrost :8080
    participant E as llm-d EPP<br/>(prod only)
    participant V as vLLM :8000
    participant G as GPU

    C->>B: POST /v1/chat/completions<br/>model: vllm-local/Qwen2.5-Coder-14B-AWQ
    alt production (llm-d)
        B->>E: route to model pool
        E->>E: prefix-aware worker pick<br/>KV cache reuse
        E->>V: forward request
    else learning (direct)
        B->>V: proxy to registered provider
    end
    V->>G: prefill + decode tokens
    G-->>V: logits
    V-->>B: SSE stream / JSON
    B-->>C: OpenAI-format response
    Note over B: logs latency, provider,<br/>fallback if vLLM down
```

**Direct to vLLM (no gateway):** skip Bifrost — client → `http://<host>:8000/v1/chat/completions`.

---

## 4. Autoscaling chain (production)

```mermaid
flowchart LR
    load["Traffic ↑"] --> q["EPP queue depth ↑"]
    q --> keda["KEDA +1 replica"]
    keda --> pend["Pending vLLM pod"]
    pend --> karp["Karpenter<br/>new GPU node"]
    karp --> run["Pod Running<br/>queue drains"]
    run --> idle["Traffic ↓"]
    idle --> keda2["KEDA -1 replica"]
    keda2 --> empty["Node empty"]
    empty --> karp2["Karpenter consolidate<br/>~1 min"]
    karp2 --> bill["Bill stops"]
```

Local equivalent: [Lab 05](labs/lab-05-keda-autoscale.md) uses `vllm:num_requests_waiting` instead of EPP queue size.

---

## 5. How clients consume the stack

```mermaid
flowchart TB
    subgraph endpoints["OpenAI-compatible endpoints"]
        direct["vLLM direct<br/>http://HOST:8000/v1"]
        bifrost["Bifrost gateway<br/>http://HOST:8080/v1<br/>or /vllm-local/v1"]
    end

    subgraph tools["Consumption tools"]
        pi["Pi harness<br/>pi CLI + models.json"]
        bfcli["Bifrost CLI<br/>@maximhq/bifrost-cli"]
        openai["OpenAI Python SDK"]
        cc["Claude Code / Continue"]
        owui["Open WebUI"]
    end

    pi -->|"custom provider"| direct
    pi -->|"custom provider"| bifrost
    bfcli -->|"auto-config agents"| bifrost
    openai --> direct
    openai --> bifrost
    cc --> bifrost
    cc --> direct
    owui --> direct
    owui --> bifrost
```

**Recommendation:** use **Bifrost** in front for fallback + logging; point **Pi** at Bifrost so one config covers self-hosted + Claude overflow.

---

## 6. Pi harness — setup

[Pi](https://github.com/earendil-works/pi) (`@earendil-works/pi-coding-agent`) is a minimal coding agent CLI. It talks to any OpenAI-compatible API via `~/.pi/agent/models.json`.

### Install

```bash
npm install -g @earendil-works/pi-coding-agent
```

### Option A — direct to vLLM (Jarvis / Modal / local)

Create `~/.pi/agent/models.json`:

```json
{
  "providers": {
    "vllm-jarvis": {
      "baseUrl": "http://<JARVIS_IP>:8000/v1",
      "api": "openai-completions",
      "apiKey": "none",
      "compat": {
        "supportsDeveloperRole": false,
        "supportsReasoningEffort": false
      },
      "models": [
        {
          "id": "Qwen2.5-Coder-14B-AWQ",
          "name": "Qwen2.5 Coder 14B AWQ",
          "contextWindow": 32768,
          "maxTokens": 8192
        }
      ]
    }
  }
}
```

Modal URL example: `"baseUrl": "https://your-workspace--llm-lab-coder-serve.modal.run/v1"`

Run:

```bash
pi --model vllm-jarvis/Qwen2.5-Coder-14B-AWQ
# or inside TUI: /model
```

Verify models load:

```bash
pi --list-models vllm-jarvis
```

### Option B — through Bifrost (fallback + one URL)

Point Pi at Bifrost instead of vLLM directly:

```json
{
  "providers": {
    "bifrost": {
      "baseUrl": "http://localhost:8080/v1",
      "api": "openai-completions",
      "apiKey": "none",
      "compat": {
        "supportsDeveloperRole": false,
        "supportsReasoningEffort": false
      },
      "models": [
        { "id": "vllm-local/Qwen2.5-Coder-14B-AWQ" }
      ]
    }
  }
}
```

Model `id` must match what Bifrost exposes — check `curl http://localhost:8080/v1/models`.

---

## 7. Bifrost CLI — setup

Bifrost CLI wires coding agents to your gateway without hand-editing env vars.

### Start the gateway

```bash
# gateway (pick one)
docker run --rm -p 8080:8080 -v "$(pwd)/bifrost-data:/app/data" maximhq/bifrost
# or: npx -y @maximhq/bifrost
```

Register vLLM ([Lab 04](labs/lab-04-bifrost.md)):

```bash
curl http://localhost:8080/api/providers \
  -H 'Content-Type: application/json' \
  -d '{
    "provider": "vllm-local",
    "keys": [{"name":"k1","value":"dummy","models":["*"],"weight":1.0}],
    "network_config": {"base_url":"http://host.docker.internal:8000","default_request_timeout_in_seconds":120},
    "custom_provider_config": {"base_provider_type":"openai","allowed_requests":{"chat_completion":true,"chat_completion_stream":true}}
  }'
```

### Launch Bifrost CLI

```bash
npx -y @maximhq/bifrost-cli
```

Interactive steps:

1. **Base URL** → `http://localhost:8080`
2. **Agent** → Pi / Claude Code / Codex / OpenCode (CLI fetches models from `/v1/models`)
3. **Model** → `vllm-local/Qwen2.5-Coder-14B-AWQ`

CLI sets `OPENAI_BASE_URL`, API key, and model for the agent automatically.

### curl through Bifrost (no CLI)

```bash
curl http://localhost:8080/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "vllm-local/Qwen2.5-Coder-14B-AWQ",
    "messages": [{"role":"user","content":"Write a KEDA ScaledObject for vLLM queue depth."}]
  }'
```

With hosted fallback when vLLM is down:

```bash
curl http://localhost:8080/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "vllm-local/Qwen2.5-Coder-14B-AWQ",
    "messages": [{"role":"user","content":"Hello"}],
    "fallbacks": ["anthropic/claude-sonnet-4-20250514"]
  }'
```

---

## 8. End-to-end: Mac + Jarvis + Bifrost + Pi

```mermaid
flowchart LR
    subgraph mac["MacBook"]
        pi["pi CLI"]
        bfcli["bifrost-cli"]
        bf["Bifrost :8080"]
    end

    subgraph jarvis["Jarvis L4 ~₹36/hr"]
        vllm["vLLM :8000<br/>Qwen2.5-Coder-14B-AWQ"]
    end

    pi --> bf
    bfcli --> bf
    bf -->|"http://JARVIS_IP:8000"| vllm
```

**Boot order:**

1. Jarvis: start vLLM ([Lab 07](labs/lab-07-jarvislabs-gpu.md))
2. Mac: `docker run ... maximhq/bifrost` + register Jarvis IP as provider
3. Mac: `pi --model bifrost/vllm-local/Qwen2.5-Coder-14B-AWQ` **or** `npx -y @maximhq/bifrost-cli`

---

## 9. Model reference (labs)

| Hugging Face ID | Quant | VRAM | Used in |
|-----------------|-------|------|---------|
| `Qwen/Qwen2.5-Coder-14B-Instruct-AWQ` | AWQ 4-bit | ~9 GB + KV | Lab 07, 08 (primary) |
| `Qwen/Qwen2.5-Coder-32B-Instruct-AWQ` | AWQ 4-bit | ~18 GB + KV | Lab 07 stretch |
| `Qwen/Qwen2.5-Coder-7B-Instruct-AWQ` | AWQ 4-bit | ~5 GB | Lab 07 fallback |
| `Qwen/Qwen2.5-0.5B-Instruct` | FP16 | CPU | Lab 01, 05 |

vLLM serve name (for clients): `--served-model-name Qwen2.5-Coder-14B-AWQ`

---

## Related docs

| Doc | Topic |
|-----|-------|
| [Lab 04 — Bifrost](labs/lab-04-bifrost.md) | Gateway UI + provider registration |
| [Lab 07 — Jarvis](labs/lab-07-jarvislabs-gpu.md) | GPU VM + vLLM Docker |
| [Lab 08 — Modal](labs/lab-08-modal-serverless.md) | Serverless vLLM |
| [Lab 05 — KEDA](labs/lab-05-keda-autoscale.md) | Autoscale on queue depth |
| [blog.md](blog.md) | Full production runbook |
