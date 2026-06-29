# DevSecOps Kubernetes Pipeline

A production-pattern DevSecOps pipeline running on Kubernetes — demonstrating security gates at the code level, image level, and cluster admission level, wired together with GitOps delivery.

Built to mirror real-world financial services platform engineering: policy enforcement at admission, automated security scanning in CI, and Git as the single source of truth for cluster state.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Developer Workstation                        │
│                            git push                                  │
└───────────────────────────────┬─────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        GitHub Actions CI                             │
│                                                                      │
│   ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐  │
│   │  CodeQL SAST    │   │  Trivy FS Scan  │   │ Build+Scan+Push │  │
│   │                 │   │                 │   │                 │  │
│   │ Scans Python    │   │ Scans repo,     │   │ Builds image    │  │
│   │ source for      │   │ Dockerfile &    │   │ Trivy image scan│  │
│   │ vulnerabilities │   │ k8s manifests   │   │ Push to GHCR    │  │
│   │                 │   │ for misconfigs  │   │ Update tag in   │  │
│   │                 │   │                 │   │ deployment.yaml │  │
│   └────────┬────────┘   └────────┬────────┘   └────────┬────────┘  │
│            │                     │                      │           │
│            └─────────────────────┘                      │           │
│                     both must pass                       │           │
│                                                          │           │
└──────────────────────────────────────────────────────────┼──────────┘
                                                           │
                         commit updated image tag back to Git
                                                           │
                                                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        GitHub Repository                             │
│                     (single source of truth)                        │
│                                                                      │
│   k8s/overlays/dev/    k8s/overlays/prod/    policies/gatekeeper/  │
└───────────────────────────────┬─────────────────────────────────────┘
                                │
                         ArgoCD watches repo
                         auto-syncs on change
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      kind Cluster (local)                            │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                  OPA Gatekeeper (admission controller)        │   │
│  │                                                               │   │
│  │  Every deployment passes through these gates before           │   │
│  │  reaching the scheduler:                                      │   │
│  │                                                               │   │
│  │  ✗ no-privileged-containers   ✗ disallow-latest-tag          │   │
│  │  ✗ require-resource-limits    ✗ require-labels               │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
│   ┌──────────────────────┐      ┌──────────────────────┐           │
│   │   namespace: dev      │      │   namespace: prod     │           │
│   │                       │      │                       │           │
│   │   devsecops-demo      │      │   devsecops-demo      │           │
│   │   replicas: 1         │      │   replicas: 2         │           │
│   │   Running          │      │      Running          │           │
│   └──────────────────────┘      └──────────────────────┘           │
│                                                                      │
│   control-plane    worker-1    worker-2                              │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Stack

| Layer | Tool | Purpose |
|---|---|---|
| Cluster | kind | Local 3-node Kubernetes cluster |
| GitOps | ArgoCD | Declarative delivery from Git to cluster |
| Manifests | Kustomize | Environment overlays (dev/prod) |
| Admission | OPA Gatekeeper | Policy enforcement at the Kubernetes API |
| SAST | CodeQL | Static analysis of Python source code |
| Image scanning | Trivy | Container image and filesystem vulnerability scanning |
| Registry | GHCR | GitHub Container Registry |
| CI | GitHub Actions | Automated pipeline on every push |
| App | Python / Flask | Sample microservice |

---

## Pipeline Stages

### Stage 1 — CodeQL SAST
Scans Python source code on every push for security vulnerabilities including injection flaws, unsafe deserialization, and hardcoded credentials. Results appear in the GitHub Security tab under Code Scanning.

### Stage 2 — Trivy Filesystem Scan
Runs two scans against the repository:
- `fs` scan — checks `requirements.txt` for vulnerable Python packages
- `config` scan — checks the Dockerfile and Kubernetes manifests for misconfigurations

Both stages must pass before the build proceeds.

### Stage 3 — Build, Scan, Push
1. Builds the Docker image
2. Scans the built image with Trivy for CRITICAL CVEs
3. Pushes the image to GitHub Container Registry tagged with the commit SHA
4. Updates `k8s/base/deployment.yaml` with the new image tag
5. Commits the change back to the repository

ArgoCD detects the manifest change and automatically deploys the new image to both `dev` and `prod` namespaces.

---

## OPA Gatekeeper Policies

Four admission policies enforce security standards on every deployment before it reaches the Kubernetes scheduler:

| Policy | Violation triggered when |
|---|---|
| `require-labels` | Deployment missing `app` or `owner` labels |
| `disallow-latest-tag` | Container image uses `:latest` or has no tag |
| `require-resource-limits` | Container missing CPU or memory limits |
| `no-privileged-containers` | Container running with `privileged: true` |

### Demo: Admission Blocked

Attempting to apply a non-compliant deployment:

```bash
kubectl apply -f bad-deployment-test.yaml -n dev
```

Gatekeeper blocks it immediately:

```
Error from server (Forbidden): admission webhook "validation.gatekeeper.sh" denied the request:
[require-resource-limits]    Container 'bad-container' is missing CPU limits.
[require-resource-limits]    Container 'bad-container' is missing memory limits.
[no-privileged-containers]   Container 'bad-container' is running as privileged. This is not allowed.
[disallow-latest-tag]        Container 'bad-container' uses ':latest' tag. Pin to a specific version.
[require-app-owner-labels]   Missing required labels: {"owner"}
```

The deployment never reaches the scheduler. This is enforcement at the API server level — the same pattern used in production to prevent misconfigured workloads from ever entering a cluster.

---

## Repository Structure

```
devsecops-k8s-pipeline/
├── .github/
│   └── workflows/
│       └── ci.yml                    # Full CI pipeline definition
├── app/
│   ├── Dockerfile                    # Non-root, pinned base image
│   ├── requirements.txt
│   └── src/
│       └── app.py                    # Flask microservice
├── k8s/
│   ├── base/                         # Shared manifests
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── kustomization.yaml
│   └── overlays/
│       ├── dev/                      # 1 replica, dev env var
│       │   └── kustomization.yaml
│       └── prod/                     # 2 replicas, prod env var
│           └── kustomization.yaml
├── policies/
│   └── gatekeeper/
│       ├── require-labels-template.yaml
│       ├── require-labels-constraint.yaml
│       ├── disallow-latest-tag-template.yaml
│       ├── disallow-latest-tag-constraint.yaml
│       ├── require-resource-limits-template.yaml
│       ├── require-resource-limits-constraint.yaml
│       ├── no-privileged-containers-template.yaml
│       └── no-privileged-containers-constraint.yaml
├── argocd/
│   └── apps/
│       ├── dev-app.yaml
│       └── prod-app.yaml
└── kind-config.yaml                  # Reproducible 3-node cluster definition
```

---

## Reproduce Locally

### Prerequisites

- [Docker Desktop](https://www.docker.com/products/docker-desktop/)
- [kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installation)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [Helm](https://helm.sh/docs/intro/install/)

### 1. Clone the repo

```bash
git clone https://github.com/alloysArtifexLabs/devsecops-k8s-pipeline.git
cd devsecops-k8s-pipeline
```

### 2. Create the kind cluster

```bash
kind create cluster --config kind-config.yaml
kubectl get nodes
```

### 3. Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl wait --for=condition=available deployment -l app.kubernetes.io/name=argocd-server -n argocd --timeout=120s
```

### 4. Install OPA Gatekeeper

```bash
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/v3.17.1/deploy/gatekeeper.yaml
kubectl wait --for=condition=available deployment -l control-plane=controller-manager -n gatekeeper-system --timeout=120s
```

Reduce memory limits for local cluster:

```bash
kubectl patch deployment gatekeeper-controller-manager -n gatekeeper-system \
  --patch '{"spec":{"template":{"spec":{"containers":[{"name":"manager","resources":{"requests":{"cpu":"50m","memory":"100Mi"},"limits":{"cpu":"500m","memory":"256Mi"}}}]}}}}'
```

### 5. Apply policies

```bash
kubectl apply -f policies/gatekeeper/
```

### 6. Deploy applications via ArgoCD

```bash
kubectl create namespace dev
kubectl create namespace prod
kubectl apply -f argocd/apps/
```

### 7. Access ArgoCD UI

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Open `https://localhost:8080`

Get the admin password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

### 8. Test Gatekeeper enforcement

```bash
kubectl apply -f bad-deployment-test.yaml -n dev
```

Observe the admission webhook deny the request with all policy violations listed.

---

## Security Design Decisions

**Non-root container:** The application runs as UID 1000 (`appuser`), enforced both in the Dockerfile and via `runAsNonRoot: true` + `runAsUser: 1000` in the pod security context.

**Pinned image tags:** All images reference specific digest-pinned or SHA tags. The `:latest` tag is blocked at admission by OPA Gatekeeper — no exceptions.

**Resource limits on all containers:** CPU and memory limits are required by policy. Without limits, a single misbehaving container can exhaust node resources. Gatekeeper blocks any deployment missing these.

**Git as the source of truth:** No manual `kubectl apply` in production flows. All changes go through Git, are validated by the CI pipeline, and are deployed by ArgoCD. Manual changes to the cluster are automatically reverted by ArgoCD's self-heal.

**Security scanning at every layer:**
- Source code — CodeQL on every push
- Dependencies — Trivy `fs` scan against `requirements.txt`
- Dockerfile — Trivy `config` scan
- Built image — Trivy image scan before push
- Runtime — OPA Gatekeeper at cluster admission

---

## Author

**Alloys Obiero Omullo**
Senior Platform & DevOps Engineer
[LinkedIn](https://www.linkedin.com/in/alloys-omullo-2a9148204) · [GitHub](https://github.com/alloysArtifexLabs)