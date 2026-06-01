# argodemo

> **Simple CI/CD demo** – a plain HTML web page deployed to **OpenShift** via **ArgoCD** (GitOps).

---

## How it works

```
Git push  →  GitHub Actions (build & push image)  →  ArgoCD syncs  →  OpenShift rolls out
```

1. A developer pushes to either `main` or `dev1` (changes under `app/`).
2. GitHub Actions builds the nginx container image and pushes it to **GitHub Container Registry** (`ghcr.io`).
3. The workflow updates the branch-specific deployment manifest and commits it back:
  - `main` → `k8s/deployment.yaml` with image tag `sha-<short_sha>`
  - `dev1` → `k8s-dev1/deployment.yaml` with image tag `dev1-sha-<short_sha>`
4. ArgoCD syncs branch-specific manifests:
  - `argodemo` app tracks `main` and deploys `k8s/` (production)
  - `argodemo-dev1` app tracks `dev1` and deploys `k8s-dev1/` (preview)
5. OpenShift rolls out each environment independently.

---

## Repository layout

```
argodemo/
├── app/
│   ├── index.html          ← The demo web page (edit this to trigger a deploy)
│   └── Dockerfile          ← nginx image that runs on port 8080 (OCP non-root)
├── k8s/
│   ├── deployment.yaml     ← Production Deployment (image tag updated by CI on main)
│   ├── service.yaml        ← ClusterIP Service on port 8080
│   └── route.yaml          ← Production Route with TLS edge termination
├── k8s-dev1/
│   ├── deployment.yaml     ← Preview Deployment (image tag updated by CI on dev1)
│   ├── service.yaml        ← Preview ClusterIP Service on port 8080
│   └── route.yaml          ← Preview Route with TLS edge termination
├── argocd/
│   ├── application.yaml    ← ArgoCD Application for production (main → k8s/)
│   └── application-dev1.yaml ← ArgoCD Application for preview (dev1 → k8s-dev1/)
└── .github/
  └── workflows/
    └── ci.yaml         ← GitHub Actions CI pipeline (main + dev1)
```

---

## One-time setup

### 1 – Prerequisites

| Tool | Version |
|------|---------|
| OpenShift cluster | 4.x |
| ArgoCD | 2.x (installed in `openshift-operators` namespace) |
| `oc` / `kubectl` | any recent |

### 2 – Register the ArgoCD Applications

```bash
oc apply -f argocd/application.yaml
oc apply -f argocd/application-dev1.yaml
```

ArgoCD will create the namespaces and deploy both tracks automatically:

- `argodemo` from `main` / `k8s/`
- `argodemo-dev1` from `dev1` / `k8s-dev1/`

### 3 – Update the Route hostnames

Edit both route manifests and replace the placeholder hostnames:

```yaml
# Production
host: argodemo.apps.<cluster-name>.<base-domain>

# Preview
host: argodemo-dev1.apps.<cluster-name>.<base-domain>
```

Commit and push – ArgoCD will pick it up.

### 4 – Enable image pull from ghcr.io (if needed)

If your cluster cannot pull from `ghcr.io` anonymously, create an image pull secret:

```bash
oc create secret docker-registry ghcr-pull-secret \
  --docker-server=ghcr.io \
  --docker-username=<your-github-username> \
  --docker-password=<your-github-token> \
  -n argodemo

oc secrets link default ghcr-pull-secret --for=pull -n argodemo

oc create secret docker-registry ghcr-pull-secret \
  --docker-server=ghcr.io \
  --docker-username=<your-github-username> \
  --docker-password=<your-github-token> \
  -n argodemo-dev1

oc secrets link default ghcr-pull-secret --for=pull -n argodemo-dev1
```

---

## Triggering deployments

### Preview deployment (`dev1`)

1. Edit `app/index.html`.
2. Commit and push to `dev1`:
   ```bash
   git add app/index.html
  git commit -m "demo: test preview update"
  git push origin dev1
   ```
3. CI builds/pushes `ghcr.io/<owner>/argodemo:dev1-sha-<short_sha>` and updates `k8s-dev1/deployment.yaml`.
4. ArgoCD app `argodemo-dev1` syncs to the preview route.

### Production deployment (`main`)

1. Merge tested changes to `main`.
2. CI builds/pushes `ghcr.io/<owner>/argodemo:sha-<short_sha>` and updates `k8s/deployment.yaml`.
3. ArgoCD app `argodemo` syncs to the production route.

---

## What each file controls

| File | Change it to… |
|------|--------------|
| `app/index.html` | Update the visible page content (triggers image rebuild) |
| `k8s/deployment.yaml` | Tune production replicas/resources/probes |
| `k8s-dev1/deployment.yaml` | Tune preview replicas/resources/probes |
| `k8s/route.yaml` and `k8s-dev1/route.yaml` | Change production/preview hostnames or TLS policy |
| `argocd/application.yaml` and `argocd/application-dev1.yaml` | Change sync policy, target namespace, or source branch |
| `.github/workflows/ci.yaml` | Customize branch-specific image tagging and manifest updates |
