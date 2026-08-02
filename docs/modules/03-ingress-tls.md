# Module 3 — Ingress & TLS (Istio + cert-manager + Gateway API)

**Goal:** understand how a request from the internet reaches a model service, encrypted and access-controlled.

## The chain

```
internet → AWS LB (IP allowlist) → Istio Gateway (TLS) → HTTPRoute → Service → Bifrost → model
```

## Gateway API, not old Ingress

The blog uses the **Gateway API** (the modern replacement for `Ingress`). Three resource types matter:

| Resource | Owned by | Says |
|----------|----------|------|
| `GatewayClass` | platform (Istio installs `istio`) | "which controller implements gateways" |
| `Gateway` | platform team | "listen on these ports/hosts with this TLS" |
| `HTTPRoute` | app team | "send `host X path Y` to `Service Z`" |

This separation is why the blog can add backend routes (Grafana, Bifrost) later without touching the Gateway itself.

## Why Istio here

Istio's Gateway implementation stands up the proxy **from the `Gateway` resource itself** — no separate ingress-gateway deployment to manage. You install `istio-base` (CRDs) + `istiod` (control plane), and the `Gateway` object does the rest.

## cert-manager for TLS

cert-manager automates Let's Encrypt certificates. Two things to know:

1. By default it only understands old `Ingress`. You must **turn on Gateway API support**:
   ```bash
   --set config.enableGatewayAPI=true
   ```
2. A `ClusterIssuer` (cluster-wide) or `Issuer` (namespaced) must exist before any cert is signed.

### HTTP-01 vs DNS-01

- **HTTP-01** (blog's choice): cert-manager serves a challenge file at `http://yourdomain/.well-known/...`; Let's Encrypt fetches it to prove you own the domain. Simpler, needs the domain publicly reachable on :80.
- **DNS-01**: prove ownership via a DNS TXT record (or import an ACM cert). Works for private/wildcard, more setup.

> **Patience warning from the blog:** ACME challenges can bounce a few times. If a cert is stuck, delete it to restart the flow:
> ```bash
> kubectl get certificates,challenges,orders -A
> ```

## Access control

The blog IP-allowlists the public load balancer with an **AWS prefix list** (`pl-...`). Only whitelisted source IPs reach the gateway at all — defense before auth.

## Local-track note

On `kind`/`minikube` you skip real TLS/DNS. Use `kubectl port-forward` or a plain HTTP `Gateway`/`HTTPRoute`. You still practice the Gateway API resource model, which is the transferable skill. Envoy AI Gateway (lab-03) also uses this exact Gateway API model.

**Next:** [Module 4 — Observability](04-observability.md)
