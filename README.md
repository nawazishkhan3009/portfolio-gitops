# portfolio-gitops

GitOps repository for deploying [Nawazish Khan's portfolio application](https://nawazishkhan.click) to Kubernetes clusters across multiple cloud providers. This repo serves as the **single source of truth** for the desired state of the portfolio app — Argo CD / Flux CD watches this repo and automatically reconciles the live cluster state to match.

---

## Repository Structure

```
portfolio-gitops/
├── charts/
│   └── portfolio/              # Helm chart for the portfolio application
│       ├── Chart.yaml          # Chart metadata (name, version, appVersion)
│       ├── values.yaml         # Default values (images, env vars, service types)
│       ├── values-azure.yaml   # Azure-specific overrides (NodePort services)
│       ├── values-gke.yaml     # GKE-specific overrides (Ingress + TLS + cert-manager)
│       ├── values-minikube.yaml# Local dev overrides (Ingress, no TLS)
│       └── templates/
│           ├── frontend-deployment.yaml
│           ├── frontend-service.yaml
│           ├── backend-deployment.yaml
│           ├── backend-service.yaml
│           ├── ingress.yaml
│           └── cluster-issuer.yaml
└── clusters/
    └── azure/
        └── helmrelease.yaml    # Flux CD HelmRelease manifest for Azure AKS
```

---

## How It Works

This repository follows a **GitOps** workflow:

1. The [portfolio-app](https://github.com/nawazishkhan3009/portfolio-app) CI pipeline builds Docker images on every push to `main` and updates the image tags in `charts/portfolio/values.yaml`.
2. Flux CD (running in each cluster) detects the change in this repo and automatically reconciles the cluster to the new desired state.
3. The Helm chart is rendered using the base `values.yaml` merged with an environment-specific overlay (e.g. `values-azure.yaml`).

```
Code Push → CI builds image → updates values.yaml → Flux detects change → cluster synced
```

---

## Helm Chart

The `charts/portfolio` Helm chart deploys two workloads:

| Component  | Container Port | Image                        |
|------------|---------------|------------------------------|
| `frontend` | `80`          | `nawnwa/portfolio-frontend`  |
| `backend`  | `8080`        | `nawnwa/portfolio-backend`   |

The backend receives three environment variables at runtime pointing to the live cluster URLs for GCP, Azure, and AWS — used to demonstrate multi-cloud deployments in the portfolio UI.

### Environment Overlays

| File                  | Target        | Ingress | TLS  | Service Type |
|-----------------------|---------------|---------|------|--------------|
| `values.yaml`         | Default/base  | —       | —    | ClusterIP    |
| `values-azure.yaml`   | Azure AKS     | ❌      | ❌   | NodePort     |
| `values-gke.yaml`     | Google GKE    | ✅      | ✅   | ClusterIP    |
| `values-minikube.yaml`| Local dev     | ✅      | ❌   | ClusterIP    |

### TLS / cert-manager (GKE)

On GKE, TLS certificates are issued automatically via [cert-manager](https://cert-manager.io/) using a `ClusterIssuer` configured for DNS-01 challenges against AWS Route 53. The domain `nawazishkhan.click` is served over HTTPS with a Let's Encrypt certificate.

---

## Clusters

### Azure (`clusters/azure/helmrelease.yaml`)

The Azure AKS cluster is managed by **Flux CD**. The `HelmRelease` manifest instructs Flux to:

- Pull this GitOps repo from the `main` branch every **1 minute**
- Render the Helm chart at `charts/portfolio` using `values.yaml` + `values-azure.yaml`
- Reconcile the release every **5 minutes**

```yaml
# clusters/azure/helmrelease.yaml (simplified)
valuesFiles:
  - charts/portfolio/values.yaml
  - charts/portfolio/values-azure.yaml
```

---

## Local Development

You can render and install the chart locally against a Minikube cluster:

```bash
# Preview rendered manifests
helm template portfolio ./charts/portfolio \
  -f charts/portfolio/values.yaml \
  -f charts/portfolio/values-minikube.yaml

# Install to Minikube
helm upgrade --install portfolio ./charts/portfolio \
  -f charts/portfolio/values.yaml \
  -f charts/portfolio/values-minikube.yaml

# Add to /etc/hosts for local ingress
echo "$(minikube ip)  portfolio.local" | sudo tee -a /etc/hosts
```

---

## Related Repositories

| Repository | Description |
|---|---|
| [`portfolio-app`](https://github.com/nawazishkhan3009/portfolio-app) | Application source code & CI pipeline |
| [`portfolio-gitops`](https://github.com/nawazishkhan3009/portfolio-gitops) | This repo — GitOps config & Helm chart |

---

## Tech Stack

![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=flat&logo=kubernetes&logoColor=white)
![Helm](https://img.shields.io/badge/Helm-0F1689?style=flat&logo=helm&logoColor=white)
![Flux CD](https://img.shields.io/badge/Flux_CD-5468FF?style=flat&logo=flux&logoColor=white)
![Azure](https://img.shields.io/badge/Azure_AKS-0078D4?style=flat&logo=microsoftazure&logoColor=white)
![GCP](https://img.shields.io/badge/GCP_GKE-4285F4?style=flat&logo=googlecloud&logoColor=white)
![cert-manager](https://img.shields.io/badge/cert--manager-00ADD8?style=flat)
