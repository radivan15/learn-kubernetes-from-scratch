# Learning Kubernetes — From Local to AWS EKS

## About This Repo

This repo documents a hands-on Kubernetes learning journey, from core concepts to cloud deployment. Each phase has a portfolio checkpoint as proof of real progress.

**Stack:**

| Layer | Tool |
|---|---|
| Local cluster | k3d (k3s running inside Docker) |
| Container runtime | Docker Desktop |
| Local container registry | Docker Hub |
| Cloud cluster | AWS EKS |
| Cloud registry | AWS ECR (public repo — 50 GB/month free) |
| App | Python (FastAPI) |
| IaC | Terraform (final phase) |
| CI/CD | GitHub Actions |
| Monitoring | Prometheus + Grafana |

---

## Learning Roadmap

### Phase 1 — Kubernetes Local: Core Concepts `[LOCAL]` ✅
> Understand core Kubernetes with hands-on practice

- [x] Install k3d and create a local cluster
- [x] Verify: `kubectl cluster-info` & `kubectl get nodes`
- [x] Pod, Node, Cluster — difference from plain Docker containers
- [x] First YAML manifest: deploy a simple Pod
- [x] Deployment & ReplicaSet
- [x] Service — ClusterIP, NodePort, LoadBalancer
- [x] Namespace — isolate dev/staging environments
- [x] ConfigMap & Secret

**Checkpoint**: nginx on local Kubernetes — Deployment + Service + ConfigMap + Secret

**What I Learned**: Kubernetes Deployment provides self-healing and zero-downtime rolling
updates out of the box — when a Pod dies, it is automatically recreated to match the
desired replica count. Coming from Docker (single-host, small scale), the key shift is
that you never deploy a bare Pod in production; a Deployment is always the right unit.
Kubernetes essentially brings orchestration to what I used to manage manually across
containers.

---

### Phase 2 — Kubernetes Local: Intermediate `[LOCAL]`
> Storage, health checks, autoscaling, observability

- [x] Persistent Volume & PVC
- [x] Startup, liveness & readiness probe
- [x] Resource requests & limits
- [x] Ingress + Nginx Ingress Controller
- [ ] In-cluster DNS
- [x] Horizontal Pod Autoscaler (HPA)
- [ ] StatefulSet — PostgreSQL with automatic PVC
- [x] Metrics Server & `kubectl top`

**Checkpoint**: Full local stack — PostgreSQL (StatefulSet) + Ingress + HPA + health checks + resource limits

**What I Learned**: _to be filled after completing this phase_

---

### Phase 3 — Monitoring & Helm `[LOCAL]`
> Monitor the cluster and package manifests with Helm

- [ ] Helm: install, upgrade
- [ ] Build a custom Helm chart — FastAPI Todo API
- [ ] Deploy app via `helm install` to the dev namespace
- [ ] Multi-environment: `values-dev.yaml` (1 replica, HPA off) + `values-prod.yaml` (3 replicas, HPA on)
- [ ] Deploy Prometheus + Grafana via Helm (`kube-prometheus-stack`)
- [ ] Scrape FastAPI metrics — expose `/metrics` + ServiceMonitor
- [ ] Build a simple Grafana dashboard

**Checkpoint**: Monitoring stack (Prometheus + Grafana) running locally + multi-environment Helm chart

**What I Learned**: _to be filled after completing this phase_

---

### Phase 4 — AWS EKS `[CLOUD — PAID]`
> Deploy a production-grade cluster in the cloud

- [ ] EKS vs self-managed Kubernetes — what AWS manages vs what you manage
- [ ] Create EKS cluster with `eksctl` — node type `t3.small`
- [ ] IAM roles & service accounts — EKS Pod Identity (new approach) or IRSA (legacy)
- [ ] AWS Load Balancer Controller
- [ ] ECR — build, push & pull images (use public repo to avoid costs)
- [ ] EBS CSI Driver — persistent storage (default StorageClass: GP3)
- [ ] Karpenter — AWS-recommended autoscaler (faster than Cluster Autoscaler)
- [ ] Secrets Manager + External Secrets Operator

**Checkpoint**: Full FastAPI stack on EKS with ECR + ALB + EBS

**What I Learned**: _to be filled after completing this phase_

---

### Phase 5 — GitOps & CI/CD `[CLOUD]`
> Automate deployments like a real company

- [ ] GitHub Actions — auto build & push to ECR on git push
- [ ] CD pipeline to EKS via GitHub Actions
- [ ] ArgoCD — GitOps: cluster auto-syncs from Git
- [ ] Multi-environment pipeline (dev/staging/prod)

**Checkpoint**: Push code → build image → deploy automatically

**What I Learned**: _to be filled after completing this phase_

---

### Phase 6 — Infrastructure as Code (Terraform) `[CLOUD]`
> All infrastructure reproducible from code

- [ ] Terraform basics — resource, variable, output, state
- [ ] VPC with Terraform
- [ ] EKS cluster with Terraform
- [ ] Terraform modules
- [ ] Remote state in S3 + DynamoDB locking
- [ ] Full stack: VPC + EKS + RDS via Terraform

**Checkpoint**: One Terraform repo — `terraform apply` builds everything, `terraform destroy` removes everything

**What I Learned**: _to be filled after completing this phase_

---

## Repo Structure

```
learn-kubernetes-from-scratch/
├── README.md
├── .gitignore
├── app/                          # FastAPI Todo API source code
│   ├── main.py                   # Endpoints: /health, /ready, /todos (CRUD)
│   ├── requirements.txt
│   └── Dockerfile                # Multi-stage build, base image python:3.13-slim
├── k8s/                          # Raw Kubernetes manifests (Phase 1 & 2)
│   ├── namespace.yaml            # Namespace: dev
│   ├── configmap.yaml            # ConfigMap: non-sensitive env vars
│   ├── secret.yaml.example       # Secret template (never commit real values!)
│   ├── deployment.yaml           # Deployment — resource limits, probes, env injection
│   ├── service.yaml              # NodePort Service
│   ├── pvc.yaml                  # PersistentVolumeClaim 100Mi
│   ├── ingress.yaml              # Nginx Ingress
│   ├── hpa.yaml                  # HPA — scale when CPU > 50%
│   ├── statefulset-postgres.yaml # PostgreSQL StatefulSet + headless service
│   └── servicemonitor.yaml       # ServiceMonitor — Prometheus scrapes /metrics
└── helm/                         # Helm chart (Phase 3)
    └── web-app/
        ├── Chart.yaml            # Chart metadata
        ├── values.yaml           # Default values — base for all environments
        ├── values-dev.yaml       # Dev overrides: 1 replica, small resources, HPA off
        ├── values-prod.yaml      # Prod overrides: 3 replicas, larger resources, HPA on
        └── templates/            # Kubernetes manifest templates
```

---

## Setup

### Prerequisites

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) — required as the container runtime for k3d
- [k3d](https://k3d.io/) — `choco install k3d` or download from the releases page
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [Helm](https://helm.sh/docs/intro/install/) — required from Phase 3 onwards

### Create Local Cluster with k3d

```powershell
# Create a 1-server + 2-agent node cluster (simulates a real multi-node cluster)
k3d cluster create dev-cluster --agents 2

# Fix kubeconfig — on Windows, host.docker.internal resolves to LAN IP, not localhost.
# k3d only binds the API server port to 127.0.0.1, so kubectl can't connect without this fix.
# Run this every time after creating a new cluster.
kubectl config set-cluster k3d-dev-cluster --server=https://127.0.0.1:$(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}' | ForEach-Object { $_ -replace '.*:(\d+)$','$1' })

# Verify
kubectl config current-context   # should show: k3d-dev-cluster
kubectl get nodes
# Expected: 3 nodes — 1 server (control-plane) + 2 agents (workers)

# Delete cluster when done (saves resources)
k3d cluster delete dev-cluster
```

### Metrics Server (required for HPA)

`metrics-server` collects real-time CPU/memory usage from each node's kubelet and exposes
it via the `metrics.k8s.io` API. Without it, `kubectl top` fails and HPA can't read CPU
utilization (shows `<unknown>`).

k3d/k3s ships with its own metrics-server disabled by default. Enable it at cluster creation:

```powershell
# Create cluster with metrics-server enabled from the start
k3d cluster create dev-cluster --agents 2 --k3s-arg "--disable=traefik@server:0"

# Then install metrics-server manually (k3s does not need --kubelet-insecure-tls)
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Verify
kubectl get deployment metrics-server -n kube-system
kubectl top nodes
kubectl top pods -n dev
```

### Deploy Kubernetes Manifests (Phase 1 & 2)

```powershell
# Apply all manifests at once
kubectl apply -f k8s/

# Verify
kubectl get all -n dev

# Access app via port-forward
kubectl port-forward service/<service-name> 8080:80 -n dev
# Open: http://localhost:8080

# Test service routing from inside the cluster
kubectl exec -it -n dev deployment/<deployment-name> -- curl http://<service-name>.<namespace>.svc.cluster.local
```

### Deploy via Helm (Phase 3)

```powershell
# First install
helm install todo-api ./helm/web-app --namespace dev

# Upgrade with dev values (1 replica, HPA off, small resources)
helm upgrade todo-api ./helm/web-app --namespace dev -f helm/web-app/values-dev.yaml

# Upgrade with prod values (3 replicas, HPA on, larger resources)
helm upgrade todo-api ./helm/web-app --namespace prod -f helm/web-app/values-prod.yaml

# Dry-run before applying
helm upgrade todo-api ./helm/web-app --namespace dev -f helm/web-app/values-dev.yaml --dry-run=client

# View upgrade history
helm history todo-api -n dev

# Rollback to previous revision
helm rollback todo-api -n dev
```

### Deploy Monitoring Stack (Phase 3)

```powershell
# Add Prometheus Community repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Create monitoring namespace
kubectl create namespace monitoring

# Install Prometheus + Grafana + Alertmanager
helm install prometheus prometheus-community/kube-prometheus-stack `
  --namespace monitoring `
  --set grafana.adminPassword="admin123" `
  --set prometheus.prometheusSpec.retention="3d"

# Verify all pods are Running
kubectl get pods -n monitoring

# Access Grafana (login: admin / admin123)
kubectl port-forward svc/prometheus-grafana 3000:80 -n monitoring
# Open: http://localhost:3000

# Access Prometheus UI
kubectl port-forward svc/prometheus-kube-prometheus-prometheus 9090:9090 -n monitoring
# Open: http://localhost:9090

# Apply ServiceMonitor so Prometheus scrapes todo-api
kubectl apply -f k8s/servicemonitor.yaml
```

---

## Conventions

- All YAML manifests must have **resource limits** and **health checks** (startup + liveness + readiness probe)
- Never use `:latest` image tag — always use a specific version (`:v1.0.0`)
- Never commit real Secret values — use `secret.yaml.example` as a template
- Each phase folder has its own README explaining how to run it
- **Always delete the k3d cluster when not in use** — `k3d cluster delete dev-cluster` to free resources

---