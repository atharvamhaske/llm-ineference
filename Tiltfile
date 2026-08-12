# Single-L4 homelab stack — run after kind cluster exists.
#
# Prereqs (once on Jarvis VM):
#   kind create cluster --name inference --config manifests/homelab/kind-gpu.yaml
#   curl -fsSL https://raw.githubusercontent.com/tilt-dev/tilt/master/scripts/install.sh | bash
#
# Then:
#   tilt up
#   tilt up -- --ingress=true   # optional nginx ingress on :80
#
# Scheduling fix: both kind nodes start tainted (control-plane + decode). Pods with
# no tolerations cannot land on either. cluster-prep removes the control-plane taint
# so Prometheus/KEDA/Bifrost schedule there; GPU workloads keep decode tolerations.

allow_k8s_contexts('kind-inference')

config.define_bool('ingress', False, 'Install ingress-nginx + Bifrost Ingress')
cfg = config.parse()

# --- cluster prep (fixes FailedScheduling on tainted nodes) ---

local_resource(
    'cluster-prep',
    cmd='''
set -e
echo "Removing control-plane NoSchedule taint (homelab pattern)..."
for n in $(kubectl get nodes -l node-role.kubernetes.io/control-plane -o name 2>/dev/null); do
  kubectl taint "$n" node-role.kubernetes.io/control-plane:NoSchedule- 2>/dev/null || true
done
WORKER=$(kubectl get nodes -l llm-d.ai/role=decode -o jsonpath='{.items[0].metadata.name}' 2>/dev/null || true)
if [ -n "$WORKER" ]; then
  echo "Ensuring decode taint on GPU worker $WORKER..."
  kubectl taint node "$WORKER" llm-d.ai/role=decode:NoSchedule --overwrite
else
  echo "WARN: no node with label llm-d.ai/role=decode — create kind cluster first"
fi
''',
    labels=['setup'],
    auto_init=True,
)

local_resource(
    'helm-repos',
    cmd='''
set -e
helm repo add nvdp https://nvidia.github.io/k8s-device-plugin 2>/dev/null || true
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts 2>/dev/null || true
helm repo add gpu-helm-charts https://nvidia.github.io/dcgm-exporter/helm-charts 2>/dev/null || true
helm repo add kedacore https://kedacore.github.io/charts 2>/dev/null || true
helm repo add bifrost https://maximhq.github.io/bifrost 2>/dev/null || true
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx 2>/dev/null || true
helm repo update
''',
    resource_deps=['cluster-prep'],
    labels=['setup'],
    auto_init=True,
)

def helm_install(name, cmd, deps):
    local_resource(
        name,
        cmd=cmd,
        resource_deps=deps,
        labels=['helm'],
        auto_init=True,
    )

helm_install('nvdp', '''
helm upgrade --install nvdp nvdp/nvidia-device-plugin \
  --namespace kube-system \
  -f tilt/values/nvdp.yaml \
  --wait --timeout 5m
''', ['helm-repos'])

helm_install('prometheus', '''
helm upgrade --install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace \
  -f tilt/values/prometheus.yaml \
  --wait --timeout 10m
''', ['helm-repos'])

helm_install('dcgm', '''
helm upgrade --install dcgm-exporter gpu-helm-charts/dcgm-exporter \
  --namespace monitoring \
  -f tilt/values/dcgm.yaml \
  --wait --timeout 5m
''', ['prometheus'])

helm_install('keda', '''
helm upgrade --install keda kedacore/keda \
  --namespace keda --create-namespace \
  --wait --timeout 5m
''', ['helm-repos'])

local_resource(
    'bifrost-secret',
    cmd='''
set -e
kubectl create namespace bifrost --dry-run=client -o yaml | kubectl apply -f -
if kubectl get secret bifrost-encryption-key -n bifrost >/dev/null 2>&1; then
  echo "bifrost-encryption-key already exists, skipping"
else
  kubectl create secret generic bifrost-encryption-key \
    --from-literal=encryption-key="$(openssl rand -base64 32)" \
    -n bifrost
fi
''',
    resource_deps=['helm-repos'],
    labels=['gateway'],
    auto_init=True,
)

helm_install('bifrost', '''
helm upgrade --install bifrost bifrost/bifrost \
  --namespace bifrost \
  -f tilt/values/bifrost.yaml \
  --wait --timeout 5m
''', ['bifrost-secret'])

if cfg.get('ingress', False):
    helm_install('ingress-nginx', '''
helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace \
  --wait --timeout 5m
''', ['helm-repos'])

# --- vLLM + KEDA ScaledObject + PodMonitor ---

k8s_yaml('manifests/homelab/vllm-coder-14b.yaml')

if cfg.get('ingress', False):
    k8s_yaml('tilt/bifrost-ingress.yaml')

k8s_resource(
    'vllm-coder-14b',
    resource_deps=['nvdp', 'prometheus', 'keda'],
    labels=['serving'],
)

k8s_resource(
    objects=['vllm-coder:service'],
    new_name='vllm-api',
    port_forwards=['8000:80'],
    resource_deps=['vllm-coder-14b'],
    labels=['serving'],
)

k8s_resource(
    objects=['kube-prometheus-stack-grafana:service'],
    new_name='grafana',
    port_forwards=['3000:80'],
    resource_deps=['prometheus'],
    labels=['observability'],
)

k8s_resource(
    objects=['kube-prometheus-stack-prometheus:service'],
    new_name='prometheus-ui',
    port_forwards=['9090:9090'],
    resource_deps=['prometheus'],
    labels=['observability'],
)

k8s_resource(
    objects=['bifrost:service'],
    new_name='bifrost-gateway',
    port_forwards=['8080:8080'],
    resource_deps=['bifrost', 'vllm-coder-14b'],
    labels=['gateway'],
)
