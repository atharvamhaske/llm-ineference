# Lab 03 — Envoy AI Gateway in front of vLLM

**Goal:** replace "the gateway" slot with Envoy AI Gateway, front your Lab 01 vLLM, and add a hosted-provider fallback.
**Runs on:** the local kind cluster (Lab 00) + vLLM (Lab 01). No GPU required for the gateway itself.

> **Version pinning:** commands use the versions current at writing. **Check the [compatibility matrix](https://aigateway.envoyproxy.io/docs/compatibility/) first** and adjust `ENVOY_GATEWAY_VERSION` / `AIGW_VERSION`.

## 1. Gateway API + GAIE CRDs

```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.3.0/standard-install.yaml
# GAIE / InferencePool CRDs (optional, for the InferencePool path)
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api-inference-extension/releases/download/v1.0.1/manifests.yaml
```

## 2. Install Envoy Gateway (with AI Gateway values)

```bash
export ENVOY_GATEWAY_VERSION=v1.8.1

helm upgrade -i eg oci://docker.io/envoyproxy/gateway-helm \
  --version ${ENVOY_GATEWAY_VERSION} \
  --namespace envoy-gateway-system --create-namespace \
  -f https://raw.githubusercontent.com/envoyproxy/ai-gateway/main/manifests/envoy-gateway-values.yaml

kubectl wait --timeout=2m -n envoy-gateway-system \
  deployment/envoy-gateway --for=condition=Available
```

## 3. Install Envoy AI Gateway (CRDs + controller)

```bash
export AIGW_VERSION=v0.7.0

helm upgrade -i envoy-ai-gateway-crd oci://docker.io/envoyproxy/ai-gateway-crds-helm \
  --version ${AIGW_VERSION} \
  -n envoy-ai-gateway-system --create-namespace

helm upgrade -i envoy-ai-gateway oci://docker.io/envoyproxy/ai-gateway-helm \
  --version ${AIGW_VERSION} \
  -n envoy-ai-gateway-system --create-namespace

kubectl wait --timeout=2m -n envoy-ai-gateway-system \
  deployment/ai-gateway-controller --for=condition=Available
```

## 4. GatewayClass + Gateway

```bash
kubectl apply -f - <<'EOF'
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata: {name: envoy-ai-gateway}
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata: {name: ai-gw, namespace: serving}
spec:
  gatewayClassName: envoy-ai-gateway
  listeners:
    - {name: http, protocol: HTTP, port: 80}
EOF
```

## 5. Point the gateway at your vLLM (self-hosted backend)

`AIServiceBackend` wraps the vLLM Service from Lab 01; `AIGatewayRoute` exposes it under the OpenAI schema.

```bash
kubectl apply -f - <<'EOF'
apiVersion: aigateway.envoyproxy.io/v1alpha1
kind: AIServiceBackend
metadata: {name: vllm-selfhosted, namespace: serving}
spec:
  schema: {name: OpenAI}
  backendRef: {name: vllm-qwen, kind: Service, port: 80}
---
apiVersion: aigateway.envoyproxy.io/v1alpha1
kind: AIGatewayRoute
metadata: {name: chat, namespace: serving}
spec:
  parentRefs: [{name: ai-gw, kind: Gateway, group: gateway.networking.k8s.io}]
  rules:
    - matches:
        - headers:
            - {type: Exact, name: x-ai-eg-model, value: qwen-local}
      backendRefs:
        - {name: vllm-selfhosted}
EOF
```

> CRD API versions/fields differ between AI Gateway releases (`v1alpha1` vs `v1beta1`). If `kubectl apply` complains, run `kubectl explain aigatewayroute` for your installed version and adjust.

## 6. Test through the gateway

```bash
kubectl -n serving port-forward svc/ai-gw-<...> 8080:80 &   # find the gateway svc name
curl http://localhost:8080/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -H 'x-ai-eg-model: qwen-local' \
  -d '{"model":"qwen-local","messages":[{"role":"user","content":"hello via envoy"}]}'
```

Same OpenAI request as Lab 01 — but now it flows **client → Envoy AI Gateway → vLLM**. The model server didn't change at all; you swapped the front door.

## 7. (Stretch) add a hosted fallback — the blog's overflow pattern

Add a second `AIServiceBackend` for a hosted provider (OpenAI/Anthropic) with a `BackendSecurityPolicy` holding the API key, then add it as a lower-priority `backendRef`. Now overflow/failure spills to the hosted model — exactly the blog's "spill to Claude on queue overflow" idea, expressed in Gateway API.

## Compare to Bifrost

After [Lab 04 — Bifrost](lab-04-bifrost.md), you'll have fronted the *same* vLLM two ways. Note what changed (only the gateway) and what didn't (the model server). That's the lesson: the gateway is a pluggable slot.

## Cleanup

```bash
kubectl delete aigatewayroute chat -n serving
kubectl delete aiservicebackend vllm-selfhosted -n serving
helm uninstall envoy-ai-gateway -n envoy-ai-gateway-system
helm uninstall eg -n envoy-gateway-system
```

**Next:** [Lab 04 — Bifrost](lab-04-bifrost.md)
