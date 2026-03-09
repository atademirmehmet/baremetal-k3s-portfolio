# mehmetatademir.com.tr — DevOps Portfolio Architecture

A minimalist, dark-themed personal portfolio web application, engineered from the ground up as a **production-grade Kubernetes playground** on bare-metal servers.

> **💡 Why Kubernetes for a static site?**
>
> Yes, deploying a personal portfolio with K3s, Helm, ArgoCD, and CI/CD pipelines is **intentional over-engineering**. This project serves as a live laboratory to demonstrate hands-on expertise in bare-metal cluster provisioning, automated GitOps workflows, and secure network routing.

---

**Created by [Mehmet Atademir](https://mehmetatademir.com.tr) — DevOps Engineer**

🌐 **Live:** [mehmetatademir.com.tr](https://mehmetatademir.com.tr)

---

## 🏗️ Architecture & Traffic Flow

```
                                         ┌────────────────────────────────────────┐
 ┌──────────┐    ┌───────────────┐       │       Bare-Metal K3s Cluster           │
 │  Client  │───▶│  Cloudflare   │──────▶│                                        │
 │ (HTTPS)  │    │  (DNS & TLS)  │       │  ┌──────────────────────────────────┐  │
 └──────────┘    └───────────────┘       │  │     Traefik Ingress Controller   │  │
                                         │  │      (L7 Reverse Proxy)          │  │
                                         │  └──────────┬───────────────────────┘  │
                                         │             │                          │
                                         │       ┌─────┴──────┐                   │
                                         │       │            │                   │
                                         │       ▼            ▼                   │
                                         │  ┌─────────┐  ┌──────────┐            │
                                         │  │Frontend │  │ Backend  │            │
                                         │  │ (Nginx) │  │ (Node.js)│            │
                                         │  │ :8080   │  │  :5000   │            │
                                         │  └─────────┘  └──────────┘            │
                                         │                                        │
                                         │  ┌──────────────────────────────────┐  │
                                         │  │          ArgoCD                   │  │
                                         │  │  (GitOps Continuous Delivery)     │  │
                                         │  └──────────────────────────────────┘  │
                                         └────────────────────────────────────────┘
```

**Request Path:** `Client → Cloudflare (TLS termination) → K3s Node → Traefik Ingress → Service → Pod`

---

## 🛠️ Tech Stack & DevOps Tooling

| Domain | Technology | Implementation Details |
|--------|-----------|----------------------|
| **Infrastructure** | Bare-Metal / K3s | Lightweight Kubernetes distribution provisioned on a dedicated Linux server. No cloud provider abstraction — full control over the node. |
| **GitOps / CD** | ArgoCD | Declarative, Git-driven continuous delivery. Auto-sync with self-healing enabled. Watches `helm/portfolio/values.yaml` for image tag changes. |
| **CI Pipeline** | GitHub Actions | Semantic version–triggered pipeline (`v*.*.*` tags). Builds, pushes to GHCR, and commits updated image tags back to the repo. |
| **Package Management** | Helm 3 | Custom chart (`helm/portfolio/`) managing Deployments, Services, Ingress, and ServiceAccount resources. |
| **Networking** | Cloudflare + Traefik | Cloudflare handles DNS resolution and TLS termination. Traefik (K3s default) acts as the in-cluster L7 reverse proxy. |
| **Container Registry** | GitHub Container Registry (GHCR) | OCI-compliant registry. Images tagged with both semantic version and `latest`. |
| **Containerization** | Docker (Multi-stage) | Frontend: Astro build → `nginx-unprivileged:alpine`. Backend: `node:20-alpine` with non-root user and built-in `HEALTHCHECK`. |
| **Security** | Pod Security Context | `runAsNonRoot: true`, dropped `ALL` capabilities, `allowPrivilegeEscalation: false`. Non-root containers across the board. |

---

## ⚙️ CI/CD Pipeline — GitOps with ArgoCD

This project implements a **zero-touch GitOps deployment strategy**. No `kubectl apply` or `helm upgrade` is run manually or from CI — ArgoCD handles all cluster-side reconciliation.

### Pipeline Flow

```
 Developer                   GitHub Actions                    ArgoCD
    │                             │                              │
    │  git tag v1.2.0             │                              │
    │  git push --tags            │                              │
    │────────────────────────────▶│                              │
    │                             │                              │
    │                     ┌───────┴────────┐                     │
    │                     │  Build Stage   │                     │
    │                     │                │                     │
    │                     │ ┌────────────┐ │                     │
    │                     │ │  Frontend  │ │                     │
    │                     │ │   Build    │ │                     │
    │                     │ └─────┬──────┘ │                     │
    │                     │       │        │                     │
    │                     │ ┌─────▼──────┐ │                     │
    │                     │ │  Push to   │ │                     │
    │                     │ │   GHCR     │ │                     │
    │                     │ └────────────┘ │                     │
    │                     │                │                     │
    │                     │ ┌────────────┐ │                     │
    │                     │ │  Backend   │ │                     │
    │                     │ │   Build    │ │                     │
    │                     │ └─────┬──────┘ │                     │
    │                     │       │        │                     │
    │                     │ ┌─────▼──────┐ │                     │
    │                     │ │  Push to   │ │                     │
    │                     │ │   GHCR     │ │                     │
    │                     │ └────────────┘ │                     │
    │                     └───────┬────────┘                     │
    │                             │                              │
    │                     ┌───────▼────────┐                     │
    │                     │ Update Stage   │                     │
    │                     │                │                     │
    │                     │ sed values.yaml│                     │
    │                     │ git commit     │                     │
    │                     │ git push main  │                     │
    │                     └───────┬────────┘                     │
    │                             │                              │
    │                             │  values.yaml changed         │
    │                             │─────────────────────────────▶│
    │                             │                              │
    │                             │                    ┌─────────┴──────────┐
    │                             │                    │  Detect drift      │
    │                             │                    │  Sync to cluster   │
    │                             │                    │  Rolling update    │
    │                             │                    │  Self-heal         │
    │                             │                    └─────────┬──────────┘
    │                             │                              │
    │                             │                    Pods running v1.2.0
```

### Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| **Tag-triggered CI** (`v*.*.*`) | Enforces semantic versioning. Prevents accidental deploys from feature branch pushes. |
| **ArgoCD over `helm upgrade` in CI** | True GitOps — the Git repo is the single source of truth. No kubeconfig or cluster credentials stored in GitHub Actions for deploy. |
| **`[skip ci]` on manifest commits** | CI updates `values.yaml` and pushes to `main`. The `[skip ci]` flag prevents infinite pipeline loops. |
| **Parallel image builds** | Frontend and backend build jobs run concurrently, reducing total pipeline time. |
| **`--atomic` rollback workflow** | Manual rollback via `workflow_dispatch` with Helm's `--atomic` flag ensures all-or-nothing upgrades. |

### Rollback Strategy

A dedicated `rollback.yml` workflow allows **manual, one-click rollback** via GitHub Actions `workflow_dispatch`:

```bash
# Triggered from GitHub UI or CLI:
gh workflow run rollback.yml -f version=v1.1.0
```

This connects directly to the K3s cluster and runs `helm upgrade` with the specified version tag and `--atomic` flag for safe rollback.

---

## 🐳 Container Strategy

### Frontend — Multi-Stage Build

```dockerfile
# Stage 1: Build (Astro SSG)
FROM node:20-alpine AS build
# → npm ci → npm run build

# Stage 2: Serve (Non-root Nginx)
FROM nginxinc/nginx-unprivileged:alpine
# → Copy dist/ to /usr/share/nginx/html
# → Expose 8080 (non-root safe)
```

**Why `nginx-unprivileged`?** Standard Nginx binds to port 80, requiring root. The unprivileged variant runs on port 8080, aligning with the `runAsNonRoot` pod security policy.

### Backend — Hardened Node.js

```dockerfile
FROM node:20-alpine
# → Production-only deps (npm install --only=production)
# → Custom non-root user (UID 1001)
# → Built-in HEALTHCHECK (wget → /api/health)
```

**Endpoints exposed:**
| Endpoint | Purpose |
|----------|---------|
| `GET /api/health` | Kubernetes **liveness** probe |
| `GET /api/ready` | Kubernetes **readiness** probe |
| `GET /api/status` | Operational status + uptime |

---

## 📦 Helm Chart Deep Dive

The custom Helm chart (`helm/portfolio/`) manages the full application lifecycle:

```
helm/portfolio/
├── Chart.yaml                    # Chart metadata (v1.0.0)
├── values.yaml                   # Environment-specific configuration
└── templates/
    ├── _helpers.tpl              # Template helpers & labels
    ├── frontend-deployment.yaml  # Frontend Deployment (1 replica)
    ├── frontend-service.yaml     # ClusterIP → :8080
    ├── backend-deployment.yaml   # Backend Deployment (2 replicas)
    ├── backend-service.yaml      # ClusterIP → :5000
    ├── ingress.yaml              # Traefik Ingress (host-based routing)
    └── serviceaccount.yaml       # Dedicated ServiceAccount
```

### Ingress Routing Rules

```yaml
# Path-based routing on mehmetatademir.com.tr
/      → frontend-service:80   # Static site
/api/* → backend-service:5000  # REST API
```

### Security Defaults (values.yaml)

```yaml
podSecurityContext:
  runAsNonRoot: true
  runAsUser: 1001
  fsGroup: 1001

securityContext:
  allowPrivilegeEscalation: false
  capabilities:
    drop:
      - ALL
```

---

## 🔄 ArgoCD Application Manifest

The `argocd/application.yaml` defines the GitOps contract between the Git repository and the K3s cluster:

```yaml
spec:
  source:
    repoURL: https://github.com/atademirmehmet/baremetal-k3s-portfolio.git
    targetRevision: HEAD
    path: helm/portfolio        # Helm chart location
  destination:
    server: https://kubernetes.default.svc
    namespace: portfolio
  syncPolicy:
    automated:
      prune: true               # Remove orphaned resources
      selfHeal: true            # Revert manual cluster changes
    syncOptions:
      - CreateNamespace=true
```

**What this means in practice:**
- ✅ Push to `main` → ArgoCD detects drift → auto-syncs to cluster
- ✅ Manual `kubectl edit` on a resource → ArgoCD reverts it (self-heal)
- ✅ Delete a template from the chart → ArgoCD removes the resource from cluster (prune)

---

## 📁 Project Structure

```
mehmetatademir/
├── .github/
│   └── workflows/
│       ├── deploy.yml               # CI: Build → Push → Update manifest
│       └── rollback.yml             # Manual rollback via workflow_dispatch
├── argocd/
│   └── application.yaml            # ArgoCD GitOps application manifest
├── frontend/                        # Static website tier (Astro + Nginx)
│   ├── src/                         # Astro source files
│   ├── public/                      # Static assets
│   ├── nginx.conf                   # Custom Nginx configuration
│   ├── Dockerfile                   # Multi-stage: Build → nginx-unprivileged
│   ├── astro.config.mjs             # Astro SSG configuration
│   ├── tailwind.config.mjs          # Tailwind CSS configuration
│   ├── package.json                 # Frontend dependencies
│   └── tsconfig.json                # TypeScript configuration
├── backend/                         # API service tier (Node.js)
│   ├── server.js                    # Express REST API (health, status, ready)
│   ├── package.json                 # Backend dependencies
│   └── Dockerfile                   # Node.js Alpine + non-root user
└── helm/
    └── portfolio/                   # Custom Kubernetes Helm Chart
        ├── Chart.yaml               # Chart metadata & maintainer info
        ├── values.yaml              # Configurable deployment parameters
        └── templates/
            ├── _helpers.tpl         # Shared template definitions
            ├── frontend-deployment.yaml
            ├── frontend-service.yaml
            ├── backend-deployment.yaml
            ├── backend-service.yaml
            ├── ingress.yaml         # Traefik routing rules
            └── serviceaccount.yaml
```

---

## 🚀 Manual Deployment (Fallback / Local Testing)

While deployments are handled by ArgoCD, the Helm chart can be deployed manually for testing:

```bash
# Install or Upgrade the release
helm upgrade --install portfolio ./helm/portfolio \
  --namespace portfolio \
  --create-namespace \
  --set frontend.image.repository=ghcr.io/atademirmehmet/baremetal-k3s-portfolio/frontend \
  --set backend.image.repository=ghcr.io/atademirmehmet/baremetal-k3s-portfolio/backend

# Check deployment status
kubectl get pods -n portfolio
kubectl get ingress -n portfolio

# View ArgoCD sync status
argocd app get portfolio

# Rollback with Helm (if needed)
helm rollback portfolio 1 -n portfolio

# Uninstall
helm uninstall portfolio -n portfolio
```

---

## 📝 Configuration Reference (Helm Values)

Edit `helm/portfolio/values.yaml` to customize deployments:

| Parameter | Default | Description |
|-----------|---------|-------------|
| `frontend.replicaCount` | `1` | Number of frontend pod replicas |
| `backend.replicaCount` | `2` | Number of backend pod replicas |
| `frontend.image.tag` | `v1.2.4` | Frontend container image tag |
| `backend.image.tag` | `v1.2.4` | Backend container image tag |
| `ingress.className` | `traefik` | Ingress controller class |
| `ingress.hosts[0].host` | `mehmetatademir.com.tr` | Primary hostname |
| `frontend.resources.limits.memory` | `128Mi` | Frontend memory limit |
| `backend.resources.limits.memory` | `256Mi` | Backend memory limit |
| `podSecurityContext.runAsNonRoot` | `true` | Enforce non-root containers |

---

## 🔧 Local Development

```bash
# Frontend (Astro dev server with hot reload)
cd frontend
npm install
npm run dev
# → http://localhost:4321

# Backend (Node.js API server)
cd backend
npm install
npm run dev
# → http://localhost:5000
# → Health: http://localhost:5000/api/health
# → Status: http://localhost:5000/api/status
```

---

## 📄 License

This project is open source and available under the [MIT License](LICENSE).
