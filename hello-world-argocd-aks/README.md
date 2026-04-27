# Hello World on AKS with Argo CD, Helm, and Kustomize

This sample is easy to follow in a book. It uses the **App of Apps** pattern: one parent `Application` that only deploys *other* `Application` objects from Git, and each child `Application` syncs a real environment.

- `charts/hello-world/` — reusable Helm chart (like a Terraform module)
- `environments/<env>/` — environment-specific values (single source of truth for workload config)
- `argocd/app-of-apps.yaml` — **parent** Application (App of Apps, layer 1)
- `argocd/apps/*.yaml` — **child** Application CRs (layer 2), one per environment you add

## Project structure

```text
hello-world-argocd-aks/
├── charts/
│   └── hello-world/
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
├── environments/
│   ├── dev/
│   │   ├── kustomization.yaml
│   │   ├── values.yaml
│   │   └── namespace.yaml
│   ├── staging/
│   └── prod/
└── argocd/
    ├── app-of-apps.yaml     ← apply this once (parent)
    └── apps/
        └── hello-world-dev.yaml  ← child Application (managed by parent from Git)
```

## Two layers (how App of Apps works)

| Layer | What | Git path | K8s `Application` name |
|-------|------|----------|-------------------------|
| 1 — Parent | Syncs a **directory of Application YAMLs** into `argocd` | `argocd/apps/` | `hello-world-app-of-apps` |
| 2 — Child | Syncs the **app manifests** (Kustomize + Helm) for one env | `environments/dev/` | `hello-world-dev` |

1. You apply **`app-of-apps.yaml`** once. Argo CD keeps the `argocd` namespace in sync with whatever `Application` files live under `argocd/apps/`.
2. The child **hello-world-dev** Application (declared in Git) is created/updated, then Argo syncs `environments/dev` into namespace `hello-world-dev`.

## Git repository

- [Ultimate-Kubernetes-Deployments-with-Argo-CD-and-GitOps](https://github.com/iheanacho-chukwu/Ultimate-Kubernetes-Deployments-with-Argo-CD-and-GitOps)
- Branch: `main`

## Bootstrap (one command)

```bash
kubectl apply -n argocd -f hello-world-argocd-aks/argocd/app-of-apps.yaml
```

Verify:

```bash
kubectl -n argocd get applications.argoproj.io
# Expect: hello-world-app-of-apps and hello-world-dev

kubectl get pods -n hello-world-dev
kubectl -n hello-world-dev port-forward svc/hello-world-hello-world 8080:80
```

Open `http://127.0.0.1:8080`.

## Optional: direct apply (teaching / debugging)

To demonstrate only the child Application without the parent, you can apply:

`argocd/apps/hello-world-dev.yaml`

In a full GitOps story, the parent (`app-of-apps.yaml`) is preferred so all apps stay Git-driven from one place.

## Add staging and prod

1. Add `environments/staging` and `environments/prod` (you already have folders and values; wire them like `dev`).
2. Add `argocd/apps/hello-world-staging.yaml` and `argocd/apps/hello-world-prod.yaml` by copying `hello-world-dev.yaml` and changing `metadata.name`, `spec.source.path`, and `spec.destination.namespace`.
3. Commit. The parent Application will pick up the new files and Argo will create the new child Applications.

## How the workload is built (child sync path)

1. Argo CD clones the repo for **hello-world-dev**.
2. It runs Kustomize on `hello-world-argocd-aks/environments/dev` with **Helm enabled**.
3. Kustomize inflates the chart at `charts/hello-world` and merges `environments/dev/values.yaml`.
4. Resources land in `hello-world-dev`.

## Local preview (optional, needs Helm on PATH)

```bash
cd hello-world-argocd-aks/environments/dev
kubectl kustomize . --enable-helm --load-restrictor LoadRestrictionsNone
```
