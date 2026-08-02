# Deep dive — Bifrost (the AI gateway the blog uses)

The blog uses **Bifrost** as its single front door. This is the piece with the **UI** you asked to explore.

## What it is

**Bifrost** ([github.com/maximhq/bifrost](https://github.com/maximhq/bifrost), by Maxim AI) is a high-performance AI gateway that unifies 20+ providers (OpenAI, Anthropic, Bedrock, Vertex, **and self-hosted vLLM**) behind **one OpenAI-compatible API**. It ships with a **web UI** for visual config, real-time request logs, analytics, and governance (virtual keys, budgets).

In the blog it does three jobs:
1. **One API** — clients (Claude Code, Open WebUI) speak one endpoint; they never touch a model Service directly.
2. **Fallback / overflow** — spill to a hosted provider (Claude) when the self-hosted queue overflows or a node is down.
3. **Cost accounting** — price the self-hosted model equal to `sonnet-5`, so the dashboard logs "what this would've cost on Claude" → the savings spreadsheet.

## The UI is the point

Unlike Envoy AI Gateway (pure CRDs), Bifrost is **dashboard-first**:

- **Model Providers** — add OpenAI/Anthropic/vLLM with clicks or API.
- **Live monitoring** — every request, latency, which model, and **fallback hits** in one pane (this is the screenshot in the blog).
- **Governance** — virtual keys (`sk-bf-*`), per-key budgets, usage caps.
- Config persists in SQLite/Postgres, or declaratively via `config.json`.

## Bifrost vs. Envoy AI Gateway

| | Bifrost | Envoy AI Gateway |
|---|---------|------------------|
| Primary interface | **Web UI** + API + file | Kubernetes CRDs |
| Setup speed | seconds (`docker run`) | more steps (Envoy GW + CRDs) |
| Ops model | app you run | Gateway API native |
| Governance/analytics UI | **built-in, rich** | via Envoy/observability stack |
| Best for | teams wanting a dashboard fast | GitOps / one Envoy for ingress+AI |

Doing both labs (03 and 04) fronting the *same* vLLM shows the gateway is a swappable slot.

## Route shape

Self-hosted vLLM is registered as a custom (OpenAI-compatible) provider; clients then call:

```
base_url = http://<bifrost>:8080/vllm      # instead of the vLLM Service directly
model    = vllm/<model-name>
auth     = x-bf-vk: sk-bf-<virtual-key>    (or Authorization: Bearer sk-bf-*)
```

## Learn it hands-on

Do [Lab 04 — Bifrost](../labs/lab-04-bifrost.md): run the UI, register your Lab 01 vLLM, add a hosted fallback, watch the logs.

## Further reading
- Gateway setup: https://docs.getbifrost.ai/quickstart/gateway/setting-up
- vLLM as a provider: https://docs.getbifrost.ai/quickstart/gateway/provider-configuration
