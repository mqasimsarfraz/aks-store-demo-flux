# AKS Store Demo — Flux GitOps Deployment

This repository contains the **Flux CD GitOps configuration** for deploying the
[AKS Store Demo](https://github.com/Azure-Samples/aks-store-demo) microservices
application onto any Kubernetes cluster.

Every commit you push to `main` is automatically reconciled to your cluster by Flux.

## Architecture

```
aks-store-demo-flux/
├── clusters/
│   └── my-cluster/
│       ├── sources/                  # Flux GitRepository source
│       │   └── aks-store-demo.yaml
│       └── kustomizations/           # Flux Kustomization
│           └── aks-store-demo.yaml
├── apps/
│   ├── base/                         # Raw K8s manifests (per-service)
│   │   ├── kustomization.yaml
│   │   ├── namespace.yaml
│   │   ├── mongodb/
│   │   ├── rabbitmq/
│   │   ├── order-service/
│   │   ├── makeline-service/
│   │   ├── product-service/
│   │   ├── store-front/
│   │   ├── store-admin/
│   │   ├── virtual-customer/
│   │   └── virtual-worker/
│   └── overlays/
│       └── dev/                      # Dev environment overlay
│           └── kustomization.yaml
└── README.md
```

## Services

| Service            | Description                              | Port |
|--------------------|------------------------------------------|------|
| store-front        | Customer-facing web UI                   | 8080 |
| store-admin        | Admin dashboard                          | 8081 |
| order-service      | Accepts and queues orders (Node.js)      | 3000 |
| makeline-service   | Processes orders from the queue (Go)     | 3001 |
| product-service    | Product catalog (Rust)                   | 3002 |
| mongodb            | Order database                           | 27017|
| rabbitmq           | Message queue                            | 5672 |
| virtual-customer   | Simulates customer orders                | —    |
| virtual-worker     | Simulates order fulfillment              | —    |

---

## Prerequisites

- A running Kubernetes cluster (AKS, EKS, kind, k3d, etc.)
- `kubectl` configured to talk to your cluster
- [Flux CLI](https://fluxcd.io/flux/installation/) installed
- A GitHub personal access token (PAT) with `repo` scope

---

## Step 1 — Create Your GitHub Repo

```bash
# Create a new repo on GitHub (or use the GitHub UI)
gh repo create aks-store-demo-flux --public --clone
cd aks-store-demo-flux

# Copy all the files from this project into it
# (or just fork/clone this repo directly)
cp -r <path-to-these-files>/* .
git add -A && git commit -m "Initial Flux GitOps structure"
git push origin main
```

## Step 2 — Bootstrap Flux on Your Cluster

```bash
export GITHUB_USER=<YOUR_GITHUB_USERNAME>
export GITHUB_TOKEN=<YOUR_GITHUB_PAT>

flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=aks-store-demo-flux \
  --branch=main \
  --path=clusters/my-cluster \
  --personal \
  --token-auth
```

This command will:
1. Install the Flux controllers in the `flux-system` namespace
2. Create a deploy key on the GitHub repo
3. Commit the Flux manifests into `clusters/my-cluster/flux-system/`
4. Start reconciling everything under `clusters/my-cluster/`

## Step 3 — Update the GitRepository URL

After bootstrap, edit `clusters/my-cluster/sources/aks-store-demo.yaml` and
replace `<YOUR_GITHUB_USERNAME>` with your actual GitHub username:

```yaml
spec:
  url: https://github.com/YOUR_USERNAME/aks-store-demo-flux
```

Commit and push. Flux will pick up the change within ~1 minute.

## Step 4 — Verify the Deployment

```bash
# Check Flux sources
flux get sources git

# Check Flux kustomizations
flux get kustomizations

# Check running pods
kubectl get pods -n pets

# Get the store-front external IP
kubectl get svc store-front -n pets
```

---

## Demo: Showing GitOps in Action

This is the key demo flow — push a commit and watch your cluster update automatically.

### Example 1: Scale the store-front

Edit `apps/overlays/dev/kustomization.yaml` and uncomment the patches section:

```yaml
patches:
  - target:
      kind: Deployment
      name: store-front
    patch: |
      - op: replace
        path: /spec/replicas
        value: 3
```

Then commit and push:

```bash
git add -A && git commit -m "Scale store-front to 3 replicas"
git push origin main
```

Watch Flux reconcile (within ~1 minute):

```bash
# Watch in real-time
flux get kustomizations --watch

# Or check pods
kubectl get pods -n pets -w
```

### Example 2: Update an image tag

Edit a deployment directly, e.g. `apps/base/store-front/deployment.yaml`:

```yaml
image: ghcr.io/azure-samples/aks-store-demo/store-front:latest
```

Commit, push, and watch the rolling update happen automatically.

### Example 3: Add a new resource

Create a new file like `apps/base/store-front/hpa.yaml`:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: store-front
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: store-front
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
```

Add it to `apps/base/store-front/kustomization.yaml`:

```yaml
resources:
  - deployment.yaml
  - service.yaml
  - hpa.yaml
```

Commit and push — Flux will create the HPA on your cluster.

---

## Useful Flux Commands

```bash
# Force an immediate reconciliation (don't wait for the interval)
flux reconcile kustomization aks-store-demo --with-source

# Suspend reconciliation (pause GitOps)
flux suspend kustomization aks-store-demo

# Resume reconciliation
flux resume kustomization aks-store-demo

# View Flux logs
flux logs --follow

# Check the health of all Flux components
flux check
```

## Cleanup

```bash
# Remove the app
flux delete kustomization aks-store-demo
kubectl delete ns pets

# Uninstall Flux entirely
flux uninstall
```

---

## How It Works

```
  You push a commit
        │
        ▼
  ┌─────────────┐     1m poll      ┌──────────────────┐
  │   GitHub     │ ◄──────────────  │  Flux Source      │
  │   main       │                  │  Controller       │
  └─────────────┘                  └──────────────────┘
                                           │
                                   detects new commit
                                           │
                                           ▼
                                   ┌──────────────────┐
                                   │  Flux Kustomize   │
                                   │  Controller       │
                                   └──────────────────┘
                                           │
                                    kustomize build
                                    + kubectl apply
                                           │
                                           ▼
                                   ┌──────────────────┐
                                   │  Your AKS         │
                                   │  Cluster          │
                                   └──────────────────┘
```

Flux polls your Git repo every **1 minute** (configurable in the GitRepository
`spec.interval`). When it detects a new commit, it runs `kustomize build` on the
path specified in the Kustomization, then applies the result to the cluster. If
you delete a resource from Git, Flux removes it from the cluster too (`prune: true`).
