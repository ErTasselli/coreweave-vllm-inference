# AI Inference on CoreWeave Kubernetes Service (CKS)

Production-ready Kubernetes project for running a GPU-backed AI inference
service — a [vLLM](https://docs.vllm.ai) server exposing an OpenAI-compatible
API — on [CoreWeave Kubernetes Service](https://docs.coreweave.com).

The same layout works for any GPU workload (training jobs, embedding
services, containerized web apps): swap the container image and args in
`kustomize/base/deployment.yaml` and keep everything else.

---

## Table of Contents

1. [Repository Structure](#1-repository-structure)
2. [Prerequisites](#2-prerequisites)
3. [Authentication — Getting a Kubeconfig](#3-authentication--getting-a-kubeconfig)
4. [Storage on CoreWeave](#4-storage-on-coreweave)
5. [GPU Node Scheduling](#5-gpu-node-scheduling)
6. [Step-by-Step Deployment Guide](#6-step-by-step-deployment-guide)
7. [Ingress & Networking](#7-ingress--networking)
8. [Monitoring & Troubleshooting](#8-monitoring--troubleshooting)
9. [Operations Cheat Sheet](#9-operations-cheat-sheet)

---

## 1. Repository Structure

```
.
├── README.md                          # You are here
├── .gitignore                         # Excludes real secrets, kubeconfigs
├── docker/
│   └── Dockerfile                     # Optional custom image on top of vllm-openai
└── kustomize/
    ├── base/                          # Environment-agnostic manifests
    │   ├── kustomization.yaml         # Binds all base resources together
    │   ├── namespace.yaml             # Dedicated namespace (ai-inference)
    │   ├── storage.yaml               # RWX PVCs: shared-nvme (models), shared-hdd (data)
    │   ├── configmap.yaml             # Non-secret runtime configuration
    │   ├── secret-template.yaml       # Secret TEMPLATE (never applied from git)
    │   ├── deployment.yaml            # vLLM server: GPU affinity, tolerations, probes
    │   ├── service.yaml               # ClusterIP (+ LoadBalancer alternative)
    │   └── ingress.yaml               # NGINX ingress + cert-manager TLS
    └── overlays/
        ├── staging/
        │   └── kustomization.yaml     # Small model, L40S, staging host/issuer
        └── production/
            ├── kustomization.yaml     # 2 replicas, prod host, prod certs
            └── patches/
                └── gpu-a100.yaml      # A100/H100 affinity + bigger CPU/RAM
```

**Design decisions**

- **Kustomize over Helm** — no templating language, plain YAML you can
  `kubectl apply -k`, and overlays keep staging/production drift explicit
  and reviewable. If you prefer Helm, the base maps 1:1 onto chart templates.
- **Stock vLLM image** — the deployment runs `vllm/vllm-openai` directly;
  `docker/Dockerfile` exists only for the day you need custom dependencies.
- **Secrets are never rendered** — `secret-template.yaml` is documentation;
  real secrets are created out-of-band (see step 3 of the deployment guide).

---

## 2. Prerequisites

| Tool | Minimum version | Install (macOS) | Purpose |
|------|-----------------|-----------------|---------|
| `kubectl` | 1.27+ | `brew install kubectl` | Applying manifests, debugging |
| `kustomize` | built into kubectl (`-k`) | — | Rendering overlays |
| `helm` | 3.12+ | `brew install helm` | Installing ingress-nginx & cert-manager |
| CoreWeave account | — | [cloud.coreweave.com](https://cloud.coreweave.com) | The cluster itself |
| CoreWeave API access token | — | Cloud Console → *API Access* | Generating kubeconfigs |

You also need:

- **A CKS cluster** created in the CoreWeave Cloud Console (or via their
  Terraform provider), in a region that stocks the GPU classes you want.
- **A Hugging Face token** (`hf_...`) if you serve gated models such as
  Llama — request model access on the HF model page first.
- **A DNS zone you control** for the ingress hostname (section 7).

Verify your tooling:

```bash
kubectl version --client && helm version
```

---

## 3. Authentication — Getting a Kubeconfig

CoreWeave clusters are managed — you never SSH into control-plane nodes.
All access goes through a kubeconfig obtained from CoreWeave:

1. Log into the [CoreWeave Cloud Console](https://cloud.coreweave.com).
2. Navigate to **API Access** (organization settings) and create an
   **API Access Token**. Store it in a password manager; it is shown once.
3. Go to your cluster's page and **Download kubeconfig**. The console
   generates a kubeconfig with an embedded token for your user.
4. Install it locally:

```bash
mkdir -p ~/.kube && mv ~/Downloads/cw-kubeconfig ~/.kube/coreweave.yaml
```

5. Point `kubectl` at it — either export for the session:

```bash
export KUBECONFIG=~/.kube/coreweave.yaml
```

   or merge it into your default config and switch contexts:

```bash
KUBECONFIG=~/.kube/config:~/.kube/coreweave.yaml kubectl config view --flatten > ~/.kube/merged && mv ~/.kube/merged ~/.kube/config
```

```bash
kubectl config use-context coreweave
```

> **Note on `/etc/rancher/k3s/k3s.yaml`:** that path only exists on
> self-managed k3s nodes (e.g. a CoreWeave bare-metal instance where you run
> k3s yourself). On managed CKS there is no node to read it from — the
> console-generated kubeconfig is the only supported path.

6. Smoke-test the connection and inspect your GPU inventory:

```bash
kubectl get nodes -L gpu.nvidia.com/class -L topology.kubernetes.io/region
```

If this prints nodes with a `CLASS` column (e.g. `L40S`, `A100_PCIE_80GB`,
`H100_NVLINK_80GB`), you're authenticated and can see the GPU fleet.

---

## 4. Storage on CoreWeave

CoreWeave provides **distributed block/file storage** consumed through
standard PVCs. The two workhorse classes:

| StorageClass | Media | Access modes | Use for |
|--------------|-------|--------------|---------|
| `shared-nvme` | NVMe | RWO, **RWX**, ROX | Model weights, HF cache, anything latency-sensitive |
| `shared-hdd` | HDD | RWO, **RWX**, ROX | Datasets, artifacts, logs — big and cheap |

Confirm the exact names on *your* cluster before deploying — legacy
CoreWeave Cloud regions suffix them (`shared-nvme-ord1`), and some CKS
clusters add VAST-based classes (`shared-vast`):

```bash
kubectl get storageclass
```

If your names differ, edit `storageClassName` in `kustomize/base/storage.yaml`.

**Why RWX matters here.** The `model-cache` PVC is `ReadWriteMany`: every
inference replica mounts the *same* volume. The first pod downloads the
weights into `HF_HOME=/models/hf-cache`; every subsequent replica (scale-up,
rescheduling, rolling update) starts from the warm cache instead of pulling
hundreds of GB from Hugging Face. `ReadOnlyMany` (ROX) is the stricter
variant — useful once weights are frozen and you want to guarantee nothing
mutates them.

**Resizing.** CoreWeave volumes support online expansion — bump
`spec.resources.requests.storage` and re-apply. Shrinking is not supported;
start conservative and grow.

---

## 5. GPU Node Scheduling

Three mechanisms cooperate to land a pod on the right GPU. This project uses
all three (see `kustomize/base/deployment.yaml`):

### 5.1 The GPU resource

GPUs are an *extended resource*. You request whole GPUs via limits — and for
GPUs, requests must equal limits:

```yaml
resources:
  limits:
    nvidia.com/gpu: "1"
```

This makes the pod schedulable **only** on nodes with free GPUs, but says
nothing about *which kind* of GPU.

### 5.2 Node labels + affinity → choosing the GPU class

CoreWeave labels every GPU node with its hardware class, primarily
**`gpu.nvidia.com/class`**. Discover what your cluster stocks:

```bash
kubectl get nodes -L gpu.nvidia.com/class --no-headers | awk '{print $NF}' | sort | uniq -c
```

Typical values: `L40S`, `A100_PCIE_40GB`, `A100_PCIE_80GB`,
`A100_NVLINK_80GB`, `H100_NVLINK_80GB`. The deployment pins a class with
node affinity:

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - key: gpu.nvidia.com/class
              operator: In
              values: [L40S]
```

Affinity with `In` + a value list beats a plain `nodeSelector` because you
can accept several acceptable classes (as `overlays/production/patches/gpu-a100.yaml`
does with A100 *and* H100 80 GB variants), which materially improves
scheduling success when one class is tight on capacity. Other useful labels
for `matchExpressions`: `topology.kubernetes.io/region` (pin a region) and
`node.coreweave.cloud/class` (broad node family).

### 5.3 Taints & tolerations → being allowed on GPU nodes

GPU nodes are tainted so ordinary CPU pods can't squat on them. Your GPU
pods must *tolerate* those taints:

```yaml
tolerations:
  - key: nvidia.com/gpu
    operator: Exists
    effect: NoSchedule
  - key: is_gpu
    operator: Exists
    effect: NoSchedule
```

`operator: Exists` tolerates any value; listing both common CoreWeave taint
keys is harmless (unmatched keys are ignored). Check what your nodes
actually carry:

```bash
kubectl get nodes -o custom-columns='NAME:.metadata.name,TAINTS:.spec.taints[*].key'
```

**Mental model:** the GPU resource gets you *counted*, affinity gets you
*placed*, tolerations get you *admitted*. Miss one and the pod sits in
`Pending` (see troubleshooting §8.1).

---

## 6. Step-by-Step Deployment Guide

### Step 0 — Sanity checks

```bash
kubectl config current-context
```

```bash
kubectl get storageclass && kubectl get nodes -L gpu.nvidia.com/class
```

Adjust `storageClassName` (base/storage.yaml) and the GPU class values
(base/deployment.yaml, production patch) to match what you see.

### Step 1 — Render and review (always look before you leap)

```bash
kubectl kustomize kustomize/overlays/staging | less
```

### Step 2 — Create namespace + storage + config

The overlay applies everything at once; apply it now and let the Deployment
sit Pending-for-secret momentarily, or apply in two passes. Simple path:

```bash
kubectl apply -k kustomize/overlays/staging
```

### Step 3 — Create the Secret (out-of-band, never from git)

```bash
kubectl create secret generic inference-secrets -n ai-inference-staging \
  --from-literal=HUGGING_FACE_HUB_TOKEN=hf_xxxxxxxxxxxxxxxx \
  --from-literal=VLLM_API_KEY=$(openssl rand -hex 32)
```

If you applied the overlay before the secret existed, restart the rollout:

```bash
kubectl rollout restart deployment/inference-server -n ai-inference-staging
```

### Step 4 — Watch the rollout

```bash
kubectl get pods -n ai-inference-staging -w
```

Expected lifecycle: `Pending` (seconds — scheduling + PVC bind) →
`ContainerCreating` (image pull, can be minutes for the multi-GB vLLM image)
→ `Running` but `0/1 Ready` (model download + load into VRAM) → `1/1 Ready`.

Follow the model load in the logs:

```bash
kubectl logs -f deployment/inference-server -n ai-inference-staging
```

### Step 5 — Verify from inside the cluster

```bash
kubectl port-forward svc/inference-server 8080:80 -n ai-inference-staging
```

```bash
curl -s http://localhost:8080/v1/models -H "Authorization: Bearer $VLLM_API_KEY"
```

```bash
curl -s http://localhost:8080/v1/chat/completions \
  -H "Authorization: Bearer $VLLM_API_KEY" -H "Content-Type: application/json" \
  -d '{"model":"qwen-2.5-1.5b","messages":[{"role":"user","content":"Say hello in five words."}]}'
```

### Step 6 — Production

Edit the hostnames in `kustomize/overlays/production/kustomization.yaml`
and the GPU class in `patches/gpu-a100.yaml`, create the prod secret in the
`ai-inference` namespace, then:

```bash
kubectl apply -k kustomize/overlays/production
```

```bash
kubectl rollout status deployment/inference-server -n ai-inference
```

### Updating

Change image tag / model / config → re-apply the overlay. The
`maxUnavailable: 0` rolling-update strategy keeps the old replica serving
until the new one is Ready (important: model loads take minutes).

```bash
kubectl apply -k kustomize/overlays/production && kubectl rollout status deployment/inference-server -n ai-inference
```

Roll back:

```bash
kubectl rollout undo deployment/inference-server -n ai-inference
```

---

## 7. Ingress & Networking

Traffic path: **Internet → CoreWeave LoadBalancer IP → ingress-nginx → Service (ClusterIP) → pods**.

### 7.1 Install ingress-nginx (once per cluster)

```bash
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace \
  --set controller.service.type=LoadBalancer \
  --set controller.service.externalTrafficPolicy=Local
```

CoreWeave assigns the controller a public IP from its LoadBalancer pool:

```bash
kubectl get svc ingress-nginx-controller -n ingress-nginx
```

### 7.2 DNS

Create an **A record** for each hostname (`inference-staging.example.com`,
`inference.example.com`) pointing at that `EXTERNAL-IP`. Wildcards
(`*.example.com`) also work if you host several services.

### 7.3 TLS with cert-manager (once per cluster)

```bash
helm upgrade --install cert-manager cert-manager \
  --repo https://charts.jetstack.io \
  --namespace cert-manager --create-namespace \
  --set crds.enabled=true
```

Create the two ClusterIssuers referenced by the manifests:

```bash
kubectl apply -f - <<'EOF'
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: you@example.com
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
      - http01:
          ingress:
            ingressClassName: nginx
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: you@example.com
    privateKeySecretRef:
      name: letsencrypt-staging-key
    solvers:
      - http01:
          ingress:
            ingressClassName: nginx
EOF
```

cert-manager then issues and auto-renews certificates for every Ingress
carrying the `cert-manager.io/cluster-issuer` annotation. Check progress:

```bash
kubectl get certificate -A
```

`READY=True` means TLS is live. (Staging uses the Let's Encrypt *staging*
issuer on purpose — untrusted certs, but no rate limits while you iterate.)

### 7.4 Alternative: raw LoadBalancer

To expose the pods without ingress/TLS termination, use the commented
`LoadBalancer` Service at the bottom of `kustomize/base/service.yaml` and
drop `ingress.yaml` from the base kustomization. You lose cert-manager TLS
and host-based routing; you gain one less moving part.

### 7.5 Streaming-friendly settings

LLM APIs stream via SSE. The ingress annotations already handle this
(`proxy-read-timeout: 3600`, `proxy-buffering: off`) — if you ever see
responses arriving in one burst at the end, buffering got re-enabled
somewhere.

---

## 8. Monitoring & Troubleshooting

### The universal first commands

```bash
kubectl get pods -n ai-inference -o wide
```

```bash
kubectl describe pod -l app.kubernetes.io/name=inference-server -n ai-inference
```

```bash
kubectl logs -f deployment/inference-server -n ai-inference --tail=200
```

```bash
kubectl get events -n ai-inference --sort-by=.lastTimestamp | tail -20
```

`describe` + `events` answer 90% of "why isn't it running" questions — read
the `Events:` section at the bottom of describe output first.

### 8.1 Pod stuck in `Pending`

| `describe` says | Cause | Fix |
|---|---|---|
| `Insufficient nvidia.com/gpu` | No free GPUs of the classes your affinity allows | Widen the affinity `values` list; check capacity: `kubectl get nodes -L gpu.nvidia.com/class` |
| `node(s) had untolerated taint` | Missing/wrong toleration | Compare node taints (`kubectl get nodes -o custom-columns='NAME:.metadata.name,TAINTS:.spec.taints[*].key'`) with the deployment's tolerations |
| `didn't match Pod's node affinity` | GPU class value doesn't exist in this cluster/region | Fix the `gpu.nvidia.com/class` values — exact spelling matters (`A100_PCIE_80GB`, not `a100`) |
| `pod has unbound immediate PersistentVolumeClaims` | PVC not bound | See 8.2 |

### 8.2 PVC stuck in `Pending`

```bash
kubectl get pvc -n ai-inference && kubectl describe pvc model-cache -n ai-inference
```

- **`storageclass ... not found`** — the class name in `storage.yaml`
  doesn't exist here. `kubectl get storageclass` and correct it. PVC
  storageClass is immutable: delete and recreate the PVC after editing.
- **Provisioning timeouts** — transient storage-backend pressure; check
  events again in a minute, then open a CoreWeave support ticket if stuck.

### 8.3 `CrashLoopBackOff` / OOM kills

```bash
kubectl logs deployment/inference-server -n ai-inference --previous
```

- **Exit code 137 / `OOMKilled`** (visible in `describe`, Last State):
  the *CPU RAM* limit is too small — weights stream through host memory
  during load. Raise `resources.limits.memory`.
- **`CUDA out of memory`** in logs: the *VRAM* is too small for the model.
  Lower `GPU_MEMORY_UTILIZATION` / `MAX_MODEL_LEN`, move to an 80 GB class,
  or raise `TENSOR_PARALLEL_SIZE` together with `nvidia.com/gpu` count.
- **NCCL/shared-memory errors** (`unhandled system error`, bus errors):
  `/dev/shm` too small — the manifests mount a 16Gi memory-backed
  `emptyDir`; raise `sizeLimit` for large tensor-parallel setups.
- **`401` from Hugging Face / gated repo errors**: bad or missing
  `HUGGING_FACE_HUB_TOKEN`, or model access not granted on the HF page.

### 8.4 Pod `Running` but never `Ready`

Almost always: the model is still loading and the startup probe is doing
its job. Watch the logs; a 70B model from a cold cache can take 10+ minutes
(download) + minutes (load). If load time exceeds the probe budget
(30 × 20 s), raise `startupProbe.failureThreshold`.

### 8.5 Ingress reachable but erroring

- **404 from nginx** — host header mismatch: DNS points at the LB but the
  Ingress `host:` doesn't match the URL used.
- **502/504** — pods not Ready (see 8.4) or streaming timeouts (see 7.5).
- **TLS warnings on staging** — expected: staging uses the Let's Encrypt
  staging CA. Production must show `READY=True` on
  `kubectl get certificate -n ai-inference`.

### 8.6 GPU visibility inside the pod

```bash
kubectl exec -it deployment/inference-server -n ai-inference -- nvidia-smi
```

You should see exactly the GPUs you requested with vLLM's memory footprint.
If `nvidia-smi` is missing or shows nothing, the pod was scheduled without
the GPU resource — re-check section 5.

### 8.7 Metrics

vLLM exports Prometheus metrics on `:8000/metrics` (already annotated for
scraping): request throughput, time-to-first-token, KV-cache usage
(`vllm:gpu_cache_usage_perc` — sustained >0.9 means you need more replicas
or VRAM). CoreWeave clusters also ship node/GPU-level metrics (DCGM) you
can chart in Grafana; pair `DCGM_FI_DEV_GPU_UTIL` with vLLM's queue depth
to decide when to scale `replicas`.

---

## 9. Operations Cheat Sheet

```bash
kubectl scale deployment/inference-server --replicas=4 -n ai-inference
```

```bash
kubectl rollout restart deployment/inference-server -n ai-inference
```

```bash
kubectl exec -it deployment/inference-server -n ai-inference -- bash
```

```bash
kubectl top pods -n ai-inference
```

```bash
kubectl delete -k kustomize/overlays/staging
```

> ⚠️ `kubectl delete -k` removes the namespace **including the PVCs** —
> model cache and datasets are gone. To tear down compute but keep storage,
> delete the Deployment/Service/Ingress individually.

---

*Questions to answer before going live: Is the GPU class available in your
region? Is the PVC big enough for every model you'll serve? Does your DNS
TTL let you fail over quickly? Did you rotate the placeholder API keys?*
