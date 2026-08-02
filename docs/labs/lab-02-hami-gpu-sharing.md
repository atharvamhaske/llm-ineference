# Lab 02 — HAMi: run two pods on one GPU

**Goal:** prove the blog's "GPUs can't be shared" rule is a *default*, not a law — split one physical GPU between two pods.
**Requires:** a node with a **real NVIDIA GPU**, driver installed, and the NVIDIA container runtime. (Cannot be done on CPU-only kind.)

> If you don't have a GPU: read [the HAMi writeup](../alternatives/hami-gpu-sharing.md) and skip the run. The concepts transfer.

## 0. Where to get a GPU cheaply

- A local machine with any NVIDIA card (even a 12 GB consumer GPU works — split it into 3× 4 GB).
- A single small cloud GPU VM (e.g. one `g6.xlarge` / a Lambda/RunPod instance) with k8s (`k3s` is easy).

## 1. Prereqs on the GPU node

```bash
nvidia-smi                              # driver works
# NVIDIA container toolkit + containerd/docker configured for the nvidia runtime
```

Make the `nvidia` runtime the default for containerd (k3s/kubeadm), then restart. (HAMi needs to inject into GPU containers.)

## 2. Install HAMi (replaces the vanilla device plugin)

```bash
helm repo add hami-charts https://project-hami.github.io/HAMi/
helm repo update

# deviceSplitCount = max pods per physical GPU (default 10)
helm install hami hami-charts/hami \
  -n kube-system \
  --set devicePlugin.deviceSplitCount=4
```

Check both pods are Running:

```bash
kubectl -n kube-system get pods | grep hami
# hami-device-plugin-...   Running
# hami-scheduler-...       Running
```

Confirm the node now advertises the vGPU resources:

```bash
kubectl get node <gpu-node> -o jsonpath='{.status.allocatable}' | tr ',' '\n' | grep nvidia
# nvidia.com/gpu, nvidia.com/gpumem, nvidia.com/gpucores
```

## 3. Deploy TWO pods, each claiming a 4 GB slice of the SAME card

```bash
for name in share-a share-b; do
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata: {name: $name}
spec:
  restartPolicy: Never
  containers:
    - name: app
      image: nvidia/cuda:12.4.1-base-ubuntu22.04
      command: ["bash","-c","nvidia-smi; sleep 3600"]
      resources:
        limits:
          nvidia.com/gpu: 1
          nvidia.com/gpumem: 4000      # 4000 MiB slice
          nvidia.com/gpucores: 30      # 30% of compute
EOF
done
```

## 4. Prove they share one physical GPU

```bash
kubectl get pods -o wide            # both Running, both on the same node
kubectl exec share-a -- nvidia-smi  # reports ~4000 MiB total, not the full card
```

The magic line in the logs:

```
[HAMI-core Msg(...)]: Initializing.....
...
| 0  Tesla ...  | 0MiB / 4000MiB |   # <-- the container sees only its 4 GB slice
```

Two containers, one card, each fenced to its slice. **This is the exact thing the blog said couldn't be done.**

## 5. Watch the scheduler enforce the limit

With `deviceSplitCount=4`, apply a 5th pod requesting the same card — it stays `Pending` because the card's slots/memory are exhausted. That's HAMi's scheduler refusing to overcommit.

## 6. (Optional) share a real model server

Swap the CUDA image for vLLM with a tiny model and `nvidia.com/gpumem: 6000`. Run two different small models on one card — a mini version of "many small models per GPU."

## Cleanup

```bash
kubectl delete pod share-a share-b --ignore-not-found
helm uninstall hami -n kube-system   # restore the vanilla plugin afterwards if needed
```

## What you learned

- The vanilla device plugin's "1 pod = 1 whole GPU" is a **policy**, swappable.
- `gpumem` / `gpucores` give sub-GPU granularity → the cost lever the blog couldn't use.
- Software isolation ≠ hardware isolation (MIG) — good for packing small workloads, not for a single card-saturating production model.

**Back to:** [HAMi writeup](../alternatives/hami-gpu-sharing.md) · **Next:** [Lab 03 — Envoy AI Gateway](lab-03-envoy-ai-gateway.md)
