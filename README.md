# Learning Kubernetes — From Local to AWS EKS

Hands-on Kubernetes learning journey, from core concepts to cloud deployment.

> Status: **Phase 1 — Core Concepts** (in progress) · Phase 0 done

---

## What it does

By the end of this project:

- An app (built, scanned and pushed to GHCR from a separate repo) is packaged with Helm and deployed through GitOps (ArgoCD)
- The same manifests run on a local k3d cluster and on AWS EKS — traffic routed through Gateway API, not cloud-specific ingress
- Security is enforced by the cluster, not by convention: RBAC, default-deny NetworkPolicy, Pod Security Standards, and admission policies
- All infrastructure — from VPC to EKS — is reproducible with Terraform, and the app is observable with Prometheus + Grafana

---

## Why I'm building it

6 years of SRE/Cloud Engineering background, returning to work after a 5-year break. This project rebuilds hands-on depth on the current Kubernetes ecosystem — Gateway API, EKS Pod Identity, GitOps, admission policy — phase by phase. The roadmap deliberately mirrors what a real SRE/Cloud Engineer role needs in production: RBAC, NetworkPolicy, autoscaling, backup/restore, IaC — not just toy manifests.

---

## Stack

| Layer | Tool |
|---|---|
| Local cluster | k3d (k3s running inside Docker) |
| Container runtime | Docker Desktop |
| Local container registry | k3d registry (declared in `k3d-config.yaml`) |
| Cloud cluster | AWS EKS |
| Cloud registry | GitHub Container Registry (GHCR) |
| Sample app | `traefik/whoami` (real app lives in a separate repo) |
| Package manager | Helm |
| Traffic routing | Gateway API (Envoy Gateway) |
| Policy | Pod Security Standards + ValidatingAdmissionPolicy |
| IaC | Terraform |
| CI | GitHub Actions + Trivy |
| CD / GitOps | ArgoCD |
| Monitoring | Prometheus + Grafana |

---

## Folder Structure

```
.
├── README.md
├── ROADMAP.md
├── k3d-config.yaml      # local cluster definition (1 server + 2 agents, registry)
├── k8s/
│   ├── pod.yaml         # first manifest: whoami Pod pulled from the local registry
│   └── deployment.yaml  # whoami Deployment, 2 replicas
└── docs/
    ├── debugging.md
    └── troubleshooting.md
```

More folders (manifests, Helm chart, Terraform) are added as the phases progress.

---

## Installation

Prerequisites (versions used): Docker Desktop 29.5.3, k3d v5.9.0, kubectl v1.34.1, Helm v4.1.4.

Create the local cluster (3 nodes, Traefik disabled, API on `127.0.0.1:6550`, local registry on port 5000):

```powershell
k3d cluster create --config k3d-config.yaml
kubectl get nodes
```

Delete it when done (the config file recreates it identically):

```powershell
k3d cluster delete dev-cluster
```

---

## Usage

Deploy the sample app as a single Pod (`apply` is idempotent — running it again reports `unchanged`):

```powershell
kubectl apply -f k8s/pod.yaml
kubectl get pods -o wide
```

```
NAME     READY   STATUS    RESTARTS   AGE   IP          NODE                       NOMINATED NODE   READINESS GATES
whoami   1/1     Running   0          12s   10.42.1.4   k3d-dev-cluster-server-0   <none>           <none>
```

The `whoami` image is built `FROM scratch` (no shell), so `kubectl exec -- sh` fails. Debug it with an ephemeral container instead:

```powershell
kubectl debug -it whoami --image=busybox:1.37 --target=whoami
# inside: ps aux ; wget -qO- localhost:80
```

Run it as a Deployment instead (2 replicas, self-healing):

```powershell
kubectl apply -f k8s/deployment.yaml
kubectl get deploy,rs,pods --show-labels
```

```
NAME                     READY   UP-TO-DATE   AVAILABLE   AGE    LABELS
deployment.apps/whoami   2/2     2            2           4m8s   app=whoami

NAME                              DESIRED   CURRENT   READY   AGE    LABELS
replicaset.apps/whoami-bbfdb8ff   2         2         2       4m8s   app=whoami,env=dev,pod-template-hash=bbfdb8ff

NAME                        READY   STATUS    RESTARTS   AGE     LABELS
pod/whoami-bbfdb8ff-9mntf   1/1     Running   0          2m22s   app=whoami,env=dev,pod-template-hash=bbfdb8ff
pod/whoami-bbfdb8ff-z42n4   1/1     Running   0          3m9s    app=whoami,env=dev,pod-template-hash=bbfdb8ff
```

Filter by label, or quarantine a misbehaving Pod by relabeling it (the ReplicaSet releases it and starts a replacement):

```powershell
kubectl get pods -l 'env in (dev,staging)'
kubectl label pod <pod-name> app=debug --overwrite
```

---

## What I learned

- **Phase 1 — Pods & debugging:** a Pod is one "logical host" — its containers share one IP, hostname and `localhost`, so a debug container can reach the app on `localhost:80`. Pod IPs live on the cluster overlay network and aren't reachable from the laptop without `port-forward` or a Service. Minimal (`scratch`) images have no shell by design, which shrinks the attack surface; `kubectl debug` with an ephemeral container is the way in — and since ephemeral containers can't be removed, the Pod is replaced rather than repaired. Also noticed that k3s schedules workloads on the control-plane node because it isn't tainted by default.
- **Phase 1 — Labels & Selectors:** Kubernetes objects don't reference each other by name — controllers and Services select Pods by label, much like tag-based dynamic inventory in AWS. Labels are metadata, so they can change on a running Pod while most of the Pod spec can't. Changing them imperatively (`kubectl label`, `kubectl run`) is fine for experiments, but anything not in Git is config drift.
- **Phase 1 — Deployment & ReplicaSet:** a ReplicaSet only keeps the Pod count (like an ASG); a Deployment manages versions by creating a new ReplicaSet for every template change (`pod-template-hash`) and keeping the old one at 0 replicas, which is what makes rollback possible. Relabeling a Pod out of the selector orphans it — its `ownerReferences` is empty, so it must be deleted by hand, even `kubectl delete deployment` won't remove it.

---

## Conventions

- Never use `:latest` image tag — always use a specific version (`:v1.0.0`)
- Never commit real Secret values — use `secret.yaml.example` as a template

---

## Roadmap

Detailed execution roadmap is in [ROADMAP.md](ROADMAP.md).
