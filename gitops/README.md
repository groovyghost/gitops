# Platform Admin Guide — `gitops/`

This folder is owned by the **Platform / Infrastructure team**.  
It manages ArgoCD itself, cluster-level addons, and registers tenant teams into the GitOps system.

> ⚠️ **Do not modify files in this folder unless you are a platform engineer.** Changes here affect the entire cluster.

---

## How the Platform Layer Works

```
                 kubectl apply -f bootstrap/root-app.yaml
                              │
                              ▼
                    ┌─────────────────┐
                    │   root-app      │  ← App of Apps
                    │ (ArgoCD App)    │    watches gitops/apps/
                    └────────┬────────┘
                             │ manages
           ┌─────────────────┼───────────────────┐
           ▼                 ▼                   ▼
     ┌──────────┐     ┌──────────┐      ┌─────────────────┐
     │  argocd  │     │   eso    │      │  tenant-data-eng │
     │ (self-   │     │  loki    │      │  (bootstraps    │
     │ managed) │     │          │      │  eks-data-apps) │
     └──────────┘     └──────────┘      └─────────────────┘
```

---

## Folder Structure

```
gitops/
├── bootstrap/
│   ├── root-app.yaml              # ONE-TIME apply — starts the entire GitOps engine
│   ├── argo-cd/
│   │   └── kustomization.yaml     # ArgoCD install via Kustomize (change ref= to upgrade)
│   └── cluster-resources/
│       └── argocd-ns.yaml         # Pre-creates the argocd namespace
└── apps/
    ├── argocd.yaml                # Self-managed ArgoCD Application
    ├── eso.yaml                   # External Secrets Operator
    ├── loki.yaml                  # Loki (log aggregation)
    ├── loki-networkpolicy.yaml    # Network policy for Loki
    ├── platform-project.yaml      # ArgoCD Project for platform tools
    ├── platform-bootstrap-project.yaml
    └── tenant-data-eng.yaml       # Bootstraps the Data Engineering team
```

---

## Bootstrapping a Brand New Cluster

Run these steps **once** on a fresh EKS cluster:

### Step 1 — Install raw ArgoCD (one-time only)
```bash
# Apply the pre-requisite namespace first
kubectl apply -f gitops/bootstrap/cluster-resources/argocd-ns.yaml

# Install ArgoCD using Kustomize
kubectl apply -k gitops/bootstrap/argo-cd/
```

### Step 2 — Apply the Root App
This single command hands all future management over to ArgoCD:
```bash
kubectl apply -f gitops/bootstrap/root-app.yaml
```

### Step 3 — Watch it come up
```bash
kubectl get applications -n argocd -w
```
ArgoCD will now sync everything in `gitops/apps/` automatically, including the Data Engineering ApplicationSets.

---

## Upgrading ArgoCD

ArgoCD is **self-managed** — this means you upgrade it the GitOps way, never with `kubectl` directly.

1. Open [`gitops/bootstrap/argo-cd/kustomization.yaml`](bootstrap/argo-cd/kustomization.yaml).
2. Change the `?ref=` tag to the desired version:
   ```yaml
   # Before
   - https://github.com/argoproj/argo-cd/manifests/base?ref=v3.5.1
   # After
   - https://github.com/argoproj/argo-cd/manifests/base?ref=v3.6.0
   ```
3. Commit and push to `main`.
4. ArgoCD will detect the change and upgrade itself.

---

## Adding a New Platform Addon (e.g., cert-manager)

1. Create a new file in `gitops/apps/`, e.g., `cert-manager.yaml`.
2. Define an ArgoCD `Application` pointing to the Helm chart or manifests.
3. Commit and push to `main`.
4. The `root-app` will automatically discover and sync the new application.

---

## Adding a New Tenant Team

1. Create a new Application file in `gitops/apps/`, e.g., `tenant-ml-team.yaml`.
2. Point it at the tenant's GitOps repo and their `argocd/` directory (which should contain their Projects and ApplicationSets).
3. Commit and push.

---

## Key Design Decisions

| Decision | Rationale |
|---|---|
| `root-app` has `prune: false` | Prevents accidental deletion of all apps if `gitops/apps/` is temporarily broken |
| `ignoreDifferences` on Application `/status` | Prevents constant OutOfSync noise from ArgoCD's live status updates |
| ArgoCD via Kustomize (not Helm) | Upgrade = change one `?ref=` tag. No Helm release state to manage. |
| Explicit `argocd-ns.yaml` | Ensures namespace exists before ArgoCD tries to deploy into it |
