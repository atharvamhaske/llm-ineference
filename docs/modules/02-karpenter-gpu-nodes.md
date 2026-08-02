# Module 2 — Karpenter & just-in-time GPU nodes

**Goal:** understand how GPU nodes appear on demand and vanish when idle — the single biggest cost lever.

## The problem

An idle GPU node bleeds money. A `g6e.16xlarge` is ~$5/hr on-demand; leaving one running "just in case" over a weekend is real money for nothing. The fix: nodes that are created **only** when a pod needs them and destroyed **as soon as** they're idle. That's Karpenter.

## Karpenter vs. the old way

- **Cluster Autoscaler** picks from *pre-defined* node groups. Rigid.
- **Karpenter** looks at *pending pods* and provisions the cheapest node that fits, directly from EC2. Then it **consolidates** — repacks and removes underused nodes.

## The three objects

```
EC2NodeClass   →  "what kind of machine" (AMI, subnets, disk, IAM)
NodePool       →  "what's allowed + policy" (instance types, spot/on-demand, taints, limits, consolidation)
Pod            →  "I need this" (nodeSelector + tolerations + resource requests)
```

The pod's `nodeSelector`/`tolerations` must match a NodePool, or nothing schedules.

## The two ideas that carry the blog

### 1. Aggressive consolidation

```yaml
disruption: {consolidationPolicy: WhenEmptyOrUnderutilized, consolidateAfter: 1m}
```

An empty GPU node is killed after **1 minute**. This is what makes "shut down at night" automatic — no traffic → no pods → no nodes → no bill.

### 2. Spot for prefill, on-demand for decode

```yaml
# prefill: interruptible work → spot (60–70% cheaper)
- {key: karpenter.sh/capacity-type, operator: In, values: ["spot","on-demand"]}
# decode: user-facing generation → on-demand (never interrupt mid-stream)
- {key: karpenter.sh/capacity-type, operator: In, values: ["on-demand"]}
```

A killed **prefill** worker just retries (tolerable). A killed **decode** worker cuts a user off mid-sentence (not tolerable). Match the capacity type to the cost of interruption.

## Taints + tolerations + nodeSelector = exclusive affinity

GPU nodes are tainted so random CPU pods can't land on them. Only pods that explicitly tolerate the taint **and** select the label get scheduled there:

```yaml
# on the NodePool
taints:
  - {key: llm-d.ai/role, value: decode, effect: NoSchedule}
  labels: {llm-d.ai/role: decode}
# on the pod
tolerations: [{key: llm-d.ai/role, operator: Exists, effect: NoSchedule}]
nodeSelector: {llm-d.ai/role: decode}
```

> **Blog gotcha:** each model gets its own NodePool. A pod with a *missing* nodeSelector can land on the wrong GPU pool and waste an expensive card. Always pin the model to its pool.

## Cost-control checklist (AWS track)

- `consolidateAfter` short on GPU pools (`1m`–`5m`).
- Set **NodePool `limits`** (`nvidia.com/gpu: N`) so a runaway autoscaler can't spin up 50 GPUs.
- Use spot wherever interruption is safe.
- Watch for `Pending` pods that never schedule — usually region has no capacity for that instance type (this is exactly why the blog's H200 plan stalled).

## The local-track equivalent

`kind`/`minikube` have no Karpenter — nodes are fixed. You still learn the *pod side* (nodeSelector, tolerations, requests) and simulate scaling with plain replicas. See [lab-00](../labs/lab-00-local-cluster.md).

**Next:** [Module 3 — Ingress & TLS](03-ingress-tls.md)
