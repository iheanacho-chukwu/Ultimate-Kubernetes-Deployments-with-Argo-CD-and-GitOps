# GitOps example — two simple apps (dev + staging)

Small GitOps layout with **separate App-of-Apps parents per environment** so dev and staging are never mixed at the Argo CD root.

- **`charts/`** — reusable Helm charts: `web-alpha`, `web-beta`.
- **`environments/<env>/<app>/`** — env-specific values + namespace + Kustomize (Helm inflation).
- **`argocd/app-of-apps-dev.yaml`** — parent **only** for dev (syncs `argocd/apps/dev/*.yaml`).
- **`argocd/app-of-apps-staging.yaml`** — parent **only** for staging (syncs `argocd/apps/staging/*.yaml`).
- **`argocd/apps/<env>/`** — child `Application` manifests for that environment only.

Labels `gitops-example.io/environment` and `gitops-example.io/app` are set on child apps so you can filter in the Argo CD UI or with `kubectl`.

## Layout

```text
gitops-example/
├── charts/
│   ├── web-alpha/
│   └── web-beta/
├── environments/
│   ├── dev/{web-alpha,web-beta}/
│   └── staging/{web-alpha,web-beta}/
└── argocd/
    ├── app-of-apps-dev.yaml       ← parent for DEV
    ├── app-of-apps-staging.yaml   ← parent for STAGING
    └── apps/
        ├── dev/
        │   ├── web-alpha.yaml
        │   └── web-beta.yaml
        └── staging/
            ├── web-alpha.yaml
            └── web-beta.yaml
```

## Argo CD graph (two independent App-of-Apps roots)

```text
app-of-apps-dev          app-of-apps-staging
        │                        │
        ▼                        ▼
   apps/dev/*.yaml        apps/staging/*.yaml
        │                        │
        ▼                        ▼
  environments/dev/*    environments/staging/*
```

## Namespaces (workloads)

| Environment | web-alpha | web-beta |
|-------------|-----------|----------|
| **dev** | `gitops-example-web-alpha` | `gitops-example-web-beta` |
| **staging** | `gitops-example-staging-web-alpha` | `gitops-example-staging-web-beta` |

## Git repository

`https://github.com/iheanacho-chukwu/Ultimate-Kubernetes-Deployments-with-Argo-CD-and-GitOps.git` on branch **`helm`** (all `Application` manifests use the same `targetRevision` so Argo clones one branch). If you later make **`main`** the source of truth, change every `targetRevision` in `gitops-example/argocd/**/*.yaml` to `main` and push.

## Bootstrap

Apply **both** parents (order does not matter):

```bash
kubectl apply -n argocd -f gitops-example/argocd/app-of-apps-dev.yaml
kubectl apply -n argocd -f gitops-example/argocd/app-of-apps-staging.yaml
```

You should see **six** `Application` resources:

- `gitops-example-app-of-apps-dev`, `gitops-example-app-of-apps-staging` (parents)
- `gitops-example-web-alpha-dev`, `gitops-example-web-beta-dev`
- `gitops-example-web-alpha-staging`, `gitops-example-web-beta-staging`

### Migrating from the old single `app-of-apps.yaml`

If you previously applied `gitops-example/argocd/app-of-apps.yaml` (resource name `gitops-example-app-of-apps`), remove it so it does not fight the new layout:

```bash
kubectl delete -n argocd application.argoproj.io/gitops-example-app-of-apps
```

Then apply the two new parent files above.

## Try the apps

**Dev**

```bash
kubectl -n gitops-example-web-alpha port-forward svc/web-alpha-web-alpha 8080:80
kubectl -n gitops-example-web-beta port-forward svc/web-beta-web-beta 8081:80
```

**Staging**

```bash
kubectl -n gitops-example-staging-web-alpha port-forward svc/web-alpha-web-alpha 8082:80
kubectl -n gitops-example-staging-web-beta port-forward svc/web-beta-web-beta 8083:80
```

## Filter Applications by environment

```bash
kubectl -n argocd get applications.argoproj.io \
  -l gitops-example.io/environment=dev
kubectl -n argocd get applications.argoproj.io \
  -l gitops-example.io/environment=staging
```
