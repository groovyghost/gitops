# Data Engineering Apps Guide — `eks-data-apps/`

This folder is owned by the **Data Engineering team**.  
It contains everything needed to deploy and manage Data Engineering applications on Kubernetes — without writing any Kubernetes YAML.

---

## How It Works

All applications share a single **generic Helm chart** (`charts/generic-app/`).  
You only need to provide a `values.yaml` file for your application. ArgoCD discovers it automatically.

```
eks-data-apps/
├── applications/
│   ├── data-pipeline-api/    ← Your app folder
│   │   └── values.yaml       ← The ONLY file you need to create/edit
│   └── ingestion-worker/
│       └── values.yaml
├── charts/
│   └── generic-app/          ← Shared Helm chart (DO NOT MODIFY)
│       ├── Chart.yaml
│       ├── values.yaml       ← Default values — read this to see what you can override
│       └── templates/
│           ├── deployment.yaml
│           ├── service.yaml
│           ├── ingress.yaml
│           └── serviceaccount.yaml
└── argocd/                   ← Managed by Platform team (DO NOT MODIFY)
    ├── projects/
    └── applicationsets/
```

---

## Branch → Environment Mapping

| Git Branch | Environment | Kubernetes Namespace |
|---|---|---|
| `dev` | Development | `data-eng-dev` |
| `qas` | QA / Staging | `data-eng-qas` |
| `main` | Production | `data-eng-prod` |

Each branch is fully independent. To deploy to an environment, push your `values.yaml` to the corresponding branch.

---

## Onboarding a New Application

### Step 1 — Checkout the target environment branch
```bash
git checkout dev
```

### Step 2 — Create your application folder
The folder name becomes your application name in ArgoCD (e.g., `my-new-app-dev`).
```bash
mkdir -p applications/my-new-app
```

### Step 3 — Create your `values.yaml`
Copy the template below into `applications/my-new-app/values.yaml` and fill in your values:

```yaml
# applications/my-new-app/values.yaml

replicaCount: 1

image:
  repository: <your-ecr-registry>/my-new-app
  tag: "v1.0.0"
  pullPolicy: Always

service:
  type: ClusterIP
  port: 8080

ingress:
  enabled: true
  className: nginx
  host: my-new-app.dev.my-org.com
  path: /

serviceAccount:
  create: true
  annotations:
    # Required if your app needs AWS access via IRSA
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/my-new-app-dev-role

env:
  - name: LOG_LEVEL
    value: "DEBUG"
  - name: DB_HOST
    value: "dev-db.internal"

resources:
  requests:
    cpu: 100m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

> 📖 For the full list of configurable values, see [`charts/generic-app/values.yaml`](charts/generic-app/values.yaml).

### Step 4 — Commit and push
```bash
git add applications/my-new-app/values.yaml
git commit -m "feat: onboard my-new-app to dev"
git push origin dev
```

**That's it!** Within ~2 minutes ArgoCD will automatically detect the new folder and deploy your application to the `data-eng-dev` namespace.

---

## Updating an Application (e.g., New Image Tag)

```bash
git checkout dev                          # or qas / main
# Edit applications/my-app/values.yaml
#   change image.tag: "v1.0.0" → "v1.1.0"
git add applications/my-app/values.yaml
git commit -m "chore: bump my-app to v1.1.0 on dev"
git push origin dev
```

ArgoCD will detect the change and roll out the new version automatically.

---

## Available `values.yaml` Options

| Key | Default | Description |
|---|---|---|
| `replicaCount` | `1` | Number of pod replicas |
| `image.repository` | `nginx` | Container image repository |
| `image.tag` | `latest` | Container image tag |
| `image.pullPolicy` | `IfNotPresent` | Image pull policy |
| `service.type` | `ClusterIP` | Kubernetes service type |
| `service.port` | `80` | Port exposed by the service |
| `ingress.enabled` | `false` | Enable/disable Ingress |
| `ingress.className` | `""` | Ingress class (e.g., `nginx`) |
| `ingress.host` | — | Hostname for the Ingress rule |
| `ingress.path` | `/` | URL path prefix |
| `ingress.annotations` | `{}` | Extra annotations on the Ingress |
| `serviceAccount.create` | `false` | Create a dedicated ServiceAccount |
| `serviceAccount.annotations` | `{}` | Annotations (e.g., IRSA role ARN) |
| `env` | `[]` | List of `{name, value}` env vars |
| `resources` | `{}` | CPU/memory requests and limits |
| `podAnnotations` | `{}` | Extra annotations on the Pod |

---

## Troubleshooting

**My app isn't showing in ArgoCD after pushing**
- Confirm the folder is under `applications/` and contains a `values.yaml`.
- Confirm you pushed to the correct branch (`dev`, `qas`, or `main`).
- ArgoCD polls every ~3 minutes. Wait or ask a platform admin to trigger a manual refresh.

**My app is `OutOfSync` in ArgoCD**
- Compare your `values.yaml` with what's running: `kubectl describe deployment <app-name> -n data-eng-dev`.
- If you didn't intend the change, revert your `values.yaml` commit and push.

**I need a Kubernetes resource type not in the generic chart (e.g., CronJob, StatefulSet)**
- Raise a request with the Platform team to add support to `charts/generic-app/`.
- Do **not** add raw Kubernetes manifests directly — everything must go through the Helm chart.

**I need environment variables from AWS Secrets Manager / SSM**
- The cluster has External Secrets Operator (ESO) installed.
- Ask the Platform team to create an `ExternalSecret` resource for your app, or request a template addition to the generic chart.
