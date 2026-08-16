# GitOps Monorepo

This is the **single source of truth** for all Kubernetes workloads running on EKS.  
The repository is split into two distinct areas of concern — one for the **platform team** and one for the **data engineering team**.

---

## Repository Structure

```
gitops/
├── gitops/                   # 🔧 PLATFORM ADMINS — cluster admin, ArgoCD, addons
│   ├── bootstrap/            # One-time cluster bootstrapping scripts & manifests
│   │   ├── root-app.yaml     # THE file you kubectl apply once to start everything
│   │   ├── argo-cd/          # Kustomize-based ArgoCD self-managed install
│   │   └── cluster-resources/# Pre-requisite cluster resources (namespaces, etc.)
│   └── apps/                 # All platform-level ArgoCD Applications
│       ├── argocd.yaml       # Self-managed ArgoCD install
│       ├── eso.yaml          # External Secrets Operator
│       ├── loki.yaml         # Loki log aggregation
│       └── tenant-data-eng.yaml  # Bootstraps the Data Eng team's apps
│
└── eks-data-apps/            # 📦 DATA ENGINEERS — application deployments
    ├── applications/         # One folder per app, containing values.yaml
    │   └── data-pipeline-api/
    │       └── values.yaml
    ├── argocd/               # ArgoCD Projects & ApplicationSets (managed by platform)
    │   ├── projects/
    │   └── applicationsets/
    └── charts/
        └── generic-app/      # Shared Helm chart used by ALL data apps
```

---

## Who Does What?

| Role | Folder | Responsibility |
|---|---|---|
| **Platform / Infra Admin** | `gitops/` | Bootstrap cluster, manage ArgoCD, manage addons (ESO, Loki), onboard new tenants |
| **Data Engineer** | `eks-data-apps/applications/` | Deploy and configure their own applications via `values.yaml` files |

---

## Quick Links

- 🔧 **Platform Admin Guide** → [`gitops/README.md`](gitops/README.md)
- 📦 **Developer Onboarding Guide** → [`eks-data-apps/README.md`](eks-data-apps/README.md)

---

## Branch Strategy

| Branch | Environment | Cluster Namespace |
|---|---|---|
| `dev` | Development | `data-eng-dev` |
| `qas` | QA / Staging | `data-eng-qas` |
| `main` | Production | `data-eng-prod` |

> **Note:** `gitops/` (the platform layer) always lives on `main`. The branch strategy only applies to application configuration in `eks-data-apps/applications/`.
