# argodemo

> **Simple CI/CD demo** – a plain HTML web page deployed to **OpenShift** via **ArgoCD** (GitOps).

---

## How it works

```
Git push  →  GitHub Actions (build & push image)  →  ArgoCD syncs  →  OpenShift rolls out
```

1. A developer pushes to the `main` branch.
2. The GitHub Actions workflow builds the nginx container image and pushes it to **GitHub Container Registry** (`ghcr.io`).
3. The workflow also bumps the image tag inside `k8s/deployment.yaml` and commits it back.
4. **ArgoCD** detects the new commit in `k8s/` and automatically syncs the manifests to the OCP cluster.
5. OpenShift performs a rolling update – the new pod serves the updated page.

---

## Repository layout

```
argodemo/
├── app/
│   ├── index.html          ← The demo web page (edit this to trigger a deploy)
│   └── Dockerfile          ← nginx image that runs on port 8080 (OCP non-root)
├── k8s/
│   ├── namespace.yaml      ← argodemo namespace
│   ├── deployment.yaml     ← Deployment (image tag updated by CI)
│   ├── service.yaml        ← ClusterIP Service on port 8080
│   └── route.yaml          ← OpenShift Route with TLS edge termination
├── argocd/
│   └── application.yaml    ← ArgoCD Application pointing at k8s/
└── .github/
    └── workflows/
        └── ci.yaml         ← GitHub Actions CI pipeline
```

---

## One-time setup

### 1 – Prerequisites

| Tool | Version |
|------|---------|
| OpenShift cluster | 4.x |
| ArgoCD | 2.x (installed in `argocd` namespace) |
| `oc` / `kubectl` | any recent |

### 2 – Register the ArgoCD Application

```bash
oc apply -f argocd/application.yaml
```

ArgoCD will create the `argodemo` namespace and deploy all manifests in `k8s/` automatically.

### 3 – Update the Route hostname

Edit `k8s/route.yaml` and replace the placeholder hostname:

```yaml
host: argodemo.apps.<cluster-name>.<base-domain>
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
```

---

## Triggering a demo deployment

The fastest way to see a full CI/CD cycle:

1. **Edit** `app/index.html` – change the version badge, heading, or any visible text.
2. **Commit and push**:
   ```bash
   git add app/index.html
   git commit -m "demo: bump version to v1.0.1"
   git push
   ```
3. Watch the **GitHub Actions** run build and push the image.
4. Open the **ArgoCD UI** – the app will show *OutOfSync* briefly, then sync automatically.
5. Reload the app URL – you will see the updated page.

---

## What each file controls

| File | Change it to… |
|------|--------------|
| `app/index.html` | Update the visible page content (triggers image rebuild) |
| `k8s/deployment.yaml` | Scale replicas, adjust CPU/memory limits, add env vars |
| `k8s/route.yaml` | Change hostname or TLS policy |
| `argocd/application.yaml` | Change sync policy, target namespace, or source branch |
| `.github/workflows/ci.yaml` | Customise the CI pipeline (registry, extra steps, etc.) |
