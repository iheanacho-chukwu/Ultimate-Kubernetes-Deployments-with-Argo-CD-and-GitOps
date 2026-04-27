# Hello World — AKS + Argo CD (multi-environment)

A minimal project that shows **three ways** to deploy the same app to three
environments (dev / staging / prod) on AKS using Argo CD.

---

## Project layout

```
claude-argocd/
│
├── helm/hello-world/              # APPROACH 1 — Helm chart
│   ├── Chart.yaml
│   ├── values.yaml                # shared defaults
│   ├── values-dev.yaml            # replicas=1, green page
│   ├── values-staging.yaml        # replicas=2, yellow page
│   ├── values-prod.yaml           # replicas=3, blue page
│   └── templates/
│       ├── _helpers.tpl
│       ├── deployment.yaml
│       ├── service.yaml
│       └── configmap.yaml         # renders env-coloured HTML from values
│
├── kustomize/                     # APPROACH 2 — Kustomize overlays
│   ├── base/                      # plain K8s manifests (no Helm)
│   │   ├── kustomization.yaml
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── configmap.yaml
│   └── overlays/
│       ├── dev/                   # namespace=hello-world-dev, replicas=1
│       │   ├── kustomization.yaml
│       │   ├── namespace.yaml
│       │   ├── deployment-patch.yaml   # patches replicas only
│       │   └── configmap-patch.yaml    # replaces full HTML (green)
│       ├── staging/               # namespace=hello-world-staging, replicas=2
│       └── prod/                  # namespace=hello-world-prod, replicas=3
│
└── argocd/
    ├── helm/                      # APPROACH 1 — one Application per env
    │   ├── app-dev.yaml
    │   ├── app-staging.yaml
    │   └── app-prod.yaml
    ├── kustomize/                 # APPROACH 2 — one Application per env
    │   ├── app-dev.yaml
    │   ├── app-staging.yaml
    │   └── app-prod.yaml
    └── applicationset.yaml        # APPROACH 3 — one file for ALL envs (recommended)
```

---

## What each approach teaches

| Approach | File count | Key concept |
|---|---|---|
| Helm per-env | 3 values files + 3 Application YAMLs | Values files override chart defaults per environment |
| Kustomize overlays | 1 base + 3 overlays + 3 Application YAMLs | Strategic-merge patches layer on top of shared base manifests |
| ApplicationSet | 1 YAML | Generator stamps out one Application per environment automatically |

---

## Quick start

### Prerequisites

```bash
# AKS cluster running, kubectl configured, Argo CD installed
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 1. Push this repo to GitHub

Update every `repoURL:` placeholder in `argocd/` to point to your repo:

```
repoURL: https://github.com/YOUR_ORG/YOUR_REPO.git
```

### 2a. Deploy with Helm (3 separate Application YAMLs)

```bash
kubectl apply -n argocd -f argocd/helm/app-dev.yaml
kubectl apply -n argocd -f argocd/helm/app-staging.yaml
kubectl apply -n argocd -f argocd/helm/app-prod.yaml
```

Each application deploys to its own namespace (`hello-world-helm-dev`, etc.).

### 2b. Deploy with Kustomize (3 separate Application YAMLs)

```bash
kubectl apply -n argocd -f argocd/kustomize/app-dev.yaml
kubectl apply -n argocd -f argocd/kustomize/app-staging.yaml
kubectl apply -n argocd -f argocd/kustomize/app-prod.yaml
```

Each application deploys to its own namespace (`hello-world-dev`, etc.).

### 2c. Deploy with ApplicationSet (ONE file for all envs — recommended)

```bash
kubectl apply -n argocd -f argocd/applicationset.yaml
```

This single command creates **three Applications** in the Argo CD UI:
`hello-world-dev`, `hello-world-staging`, `hello-world-prod`.

---

## Verify

```bash
# Watch Argo CD sync all applications
kubectl get applications -n argocd

# Check pods per environment (kustomize approach)
kubectl get pods -n hello-world-dev
kubectl get pods -n hello-world-staging
kubectl get pods -n hello-world-prod

# Port-forward to see the coloured page
kubectl port-forward svc/hello-world 8080:80 -n hello-world-dev
# open http://localhost:8080  → green "Hello World — DEV" page

kubectl port-forward svc/hello-world 8081:80 -n hello-world-staging
# open http://localhost:8081  → yellow "Hello World — STAGING" page

kubectl port-forward svc/hello-world 8082:80 -n hello-world-prod
# open http://localhost:8082  → blue "Hello World — PRODUCTION" page
```

---

## How environment differences are expressed

| Environment | Replicas | Page colour | Mechanism |
|---|---|---|---|
| dev | 1 | Green `#d4edda` | `values-dev.yaml` / `overlays/dev` |
| staging | 2 | Yellow `#fff3cd` | `values-staging.yaml` / `overlays/staging` |
| prod | 3 | Blue `#cce5ff` | `values-prod.yaml` / `overlays/prod` |

---

## Test Kustomize output locally (without Argo CD)

```bash
# Preview what kustomize build produces for each overlay
kustomize build kustomize/overlays/dev
kustomize build kustomize/overlays/staging
kustomize build kustomize/overlays/prod

# Preview what helm template produces for each environment
helm template hello-world helm/hello-world -f helm/hello-world/values-dev.yaml
helm template hello-world helm/hello-world -f helm/hello-world/values-staging.yaml
helm template hello-world helm/hello-world -f helm/hello-world/values-prod.yaml
```

---

## Teardown

```bash
# Delete ApplicationSet (also deletes the three generated Applications)
kubectl delete -n argocd -f argocd/applicationset.yaml

# Or delete individual Applications
kubectl delete -n argocd -f argocd/kustomize/
```
