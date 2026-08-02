# Lab 04 — Bifrost: the AI gateway UI, fronting vLLM

**Goal:** run Bifrost's dashboard, register your vLLM as a provider, add a hosted fallback, and watch requests + fallback hits in the UI.
**Runs on:** your laptop with Docker. Uses the vLLM from [Lab 01](lab-01-vllm-single-gpu.md) (Path A, port 8000).

## 1. Start Bifrost (with the web UI)

```bash
# ephemeral (testing)
docker run --rm -p 8080:8080 maximhq/bifrost

# OR persistent config + logs (recommended)
docker run --rm -p 8080:8080 -v "$(pwd)/bifrost-data:/app/data" maximhq/bifrost
```

Open the dashboard:

```bash
open http://localhost:8080        # macOS  (xdg-open on Linux)
```

## 2. Make sure vLLM is reachable

From Lab 01 Path A, vLLM is on `http://localhost:8000/v1`.

> **Docker networking:** Bifrost runs in a container, so `localhost:8000` from *inside* it is the container, not your host. Use `http://host.docker.internal:8000` (macOS/Windows) as the vLLM base URL, or run both on a shared Docker network.

```bash
curl http://localhost:8000/v1/models   # confirm vLLM is up on the host
```

## 3. Register vLLM as a provider (UI)

In the dashboard:
1. **Model Providers** → **Add Provider**.
2. Choose a **custom / OpenAI-compatible** provider (name it `vllm-local`).
3. Base URL: `http://host.docker.internal:8000` · API key: `dummy` · models: `*`.
4. Save.

Or via API:

```bash
curl http://localhost:8080/api/providers \
  -H 'Content-Type: application/json' \
  -d '{
    "provider": "vllm-local",
    "keys": [{"name":"vllm-key-1","value":"dummy","models":["*"],"weight":1.0}],
    "network_config": {"base_url":"http://host.docker.internal:8000","default_request_timeout_in_seconds":60},
    "custom_provider_config": {"base_provider_type":"openai","allowed_requests":{"chat_completion":true,"chat_completion_stream":true}}
  }'
```

## 4. Call the model *through* Bifrost

```bash
curl http://localhost:8080/vllm/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "vllm-local/Qwen/Qwen2.5-0.5B-Instruct",
    "messages": [{"role":"user","content":"hello through bifrost"}]
  }'
```

Now refresh the dashboard → **you should see the request in the live logs** with latency and which provider served it. This is the blog's "single pane for who's calling what."

## 5. Add a hosted fallback (the blog's overflow pattern)

1. Add a second provider (e.g. **Anthropic** or **OpenAI**) with a real API key.
2. Configure fallback so requests spill to the hosted provider when vLLM errors/overflows.
3. Stop the vLLM container (`Ctrl-C` on Lab 01) and re-run the curl → the request should **fall back to the hosted model**, and the dashboard flags it as a fallback hit.

That's exactly the blog's "spill to Claude on queue overflow / off-hours" — you just watched it happen.

## 6. (Optional) the cost-accounting trick

In the provider/model settings, set the self-hosted model's price equal to a hosted model (e.g. `sonnet`). Every self-hosted request now logs a cost *as if* it hit the hosted API. Dashboard analytics → your savings spreadsheet ([Module 7](../modules/07-economics.md)).

## Compare: Bifrost vs. Envoy AI Gateway

You've now fronted the **same** vLLM with both gateways:

| | [Lab 03 Envoy AI GW](lab-03-envoy-ai-gateway.md) | Lab 04 Bifrost |
|---|---|---|
| How you configured it | Kubernetes CRDs | Web UI / API |
| Where it runs | in-cluster (Gateway API) | a container with a dashboard |
| Fallback | extra `backendRef` + `BackendSecurityPolicy` | click / provider config |
| Visibility | via Prometheus/Grafana | built-in live logs |

**The model server never changed.** That's the architectural lesson: the gateway is a pluggable slot.

## Deploy Bifrost on Kubernetes (like the blog)

```bash
kubectl create ns bifrost
kubectl create secret generic bifrost-encryption-key \
  --from-literal=encryption-key="$(openssl rand -base64 32)" -n bifrost
# then the bifrost helm chart (see blog §5) pointing at your in-cluster vLLM Service URL
```

**Back to:** [Bifrost writeup](../alternatives/bifrost.md) · [README learning path](../../README.md)
