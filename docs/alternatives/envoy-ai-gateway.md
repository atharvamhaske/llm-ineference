# Alternative — Envoy AI Gateway (replacing Bifrost)

> The blog uses **Bifrost** as the AI gateway. **Envoy AI Gateway** is the CNCF-aligned alternative that plugs natively into the same Gateway API / GAIE model the blog already uses for ingress.

## What it is

**Envoy AI Gateway** ([aigateway.envoyproxy.io](https://aigateway.envoyproxy.io)) is a layer on top of **Envoy Gateway** (the Envoy-based Gateway API implementation). It gives you:

- One **OpenAI-compatible** endpoint fronting many backends (self-hosted vLLM *and* hosted providers like OpenAI/Anthropic/Bedrock).
- **Provider fallback**, retries, and **token-based rate limiting** (rate-limit on *tokens*, not just requests — the right unit for LLMs).
- Native **`InferencePool`** support (Gateway API Inference Extension / GAIE) — the same CRD family the blog installs for llm-d.
- Everything as **Kubernetes CRDs** (`AIGatewayRoute`, `AIServiceBackend`, `BackendSecurityPolicy`), so it's GitOps-friendly.

## Why swap it in (as a learning exercise)

| Dimension | Bifrost (blog) | Envoy AI Gateway |
|-----------|----------------|------------------|
| Config model | Bifrost dashboard/API, Helm | Pure Gateway API CRDs |
| Ingress integration | separate gateway (Istio) + Bifrost | **same Envoy Gateway does ingress + AI routing** |
| GAIE / InferencePool | via llm-d | **first-class** |
| Rate limiting | built-in | token-aware, via Envoy |
| Best when | fast setup, nice UI | you already live in Gateway API / want one Envoy for everything |

Swapping teaches you the Gateway API deeply, and shows that "the gateway" is a **pluggable slot** in the architecture — the model servers behind it don't change at all.

## Architecture with Envoy AI Gateway

```
client → Envoy Gateway (Gateway + AIGatewayRoute)
             ├─ AIServiceBackend: self-hosted  → vLLM Service (/v1)
             └─ AIServiceBackend: hosted (OpenAI/Anthropic)  ← fallback/overflow
```

The vLLM/llm-d model servers from the blog stay exactly as they are — you only replace the front door.

## Version note (verify before installing)

Versions move fast. As of writing:
- Envoy AI Gateway **v0.7.x / v1.0.x**
- requires **Envoy Gateway v1.8.1+**, **Kubernetes v1.32+**, **Gateway API v1.5.x**

Always check the [compatibility matrix](https://aigateway.envoyproxy.io/docs/compatibility/) and pin matching versions.

## Learn it hands-on

Do [Lab 03 — Envoy AI Gateway](../labs/lab-03-envoy-ai-gateway.md). Runs on the local kind cluster; front the vLLM from Lab 01, add a hosted-provider fallback.

## Further reading
- Getting started / prerequisites: https://aigateway.envoyproxy.io/docs/getting-started/prerequisites/
- InferencePool + HTTPRoute: https://aigateway.envoyproxy.io/docs/capabilities/inference/
- llm-d + Envoy AI Gateway guide: https://github.com/llm-d/llm-d/blob/main/docs/infrastructure/gateway/envoy-ai-gateway.md
