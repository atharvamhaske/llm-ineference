# Lab 00 — A local Kubernetes cluster (free, ~5 min)

**Goal:** get a working cluster so every later lab has somewhere to run — no AWS, no bill.
**You'll learn:** kind/minikube, namespaces, the pod-scheduling model (nodeSelector/taints) that the blog relies on.

## Prereqs

```bash
kubectl version --client   # any recent v1.29+
helm version               # v3.12+
docker version             # for kind
```

Install if missing (macOS):

```bash
brew install kubectl helm kind    # or: brew install minikube
```

## Option A — kind (recommended)

```bash
kind create cluster --name llm-lab --config - <<'EOF'
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
    labels:
      node-role: gpu-sim        # we "pretend" this is a GPU node
EOF

kubectl cluster-info --context kind-llm-lab
kubectl get nodes
```

## Option B — minikube

```bash
minikube start --nodes 2 --cpus 4 --memory 8192
kubectl label node minikube-m02 node-role=gpu-sim
```

## Simulate the blog's taint/nodeSelector model

Real GPU nodes are tainted so only model pods land on them. Reproduce that locally:

```bash
# taint the "gpu" worker like the blog does
kubectl taint node <worker-node-name> llm-d.ai/role=decode:NoSchedule
kubectl label  node <worker-node-name> llm-d.ai/role=decode
```

Now a pod only lands there if it both tolerates the taint **and** selects the label:

```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata: {name: placement-test}
spec:
  tolerations: [{key: llm-d.ai/role, operator: Exists, effect: NoSchedule}]
  nodeSelector: {llm-d.ai/role: decode}
  containers:
    - {name: box, image: busybox, command: ["sleep","3600"]}
EOF

kubectl get pod placement-test -o wide     # should be on the gpu-sim node
```

Try removing `nodeSelector` or `tolerations` and re-applying — watch it go `Pending` or land elsewhere. **This is exactly the "missing nodeSelector → wrong pool" bug from the blog** (Module 2).

## Sanity: install a Helm chart

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install hello bitnami/nginx --wait
kubectl get pods
helm uninstall hello
```

## Cleanup

```bash
kind delete cluster --name llm-lab      # or: minikube delete
```

## What you just learned maps to the blog

| Local thing | Blog equivalent |
|-------------|-----------------|
| kind worker node | a Karpenter-provisioned GPU node |
| manual taint/label | NodePool taints/labels |
| Pending pod | the KEDA→Karpenter handoff signal |

**Next:** [Lab 01 — Run vLLM](lab-01-vllm-single-gpu.md)
