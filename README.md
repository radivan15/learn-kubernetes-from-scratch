# Learning Kubernetes — From Local to AWS EKS

Hands-on Kubernetes learning journey, from core concepts to cloud deployment.

> Status: in progress

---

## What it does

By the end of this project:

- A FastAPI app is packaged with Helm and deployed through GitOps (ArgoCD), with images built, scanned and pushed by GitHub Actions
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
| Local container registry | k3d registry (`k3d registry create`) |
| Cloud cluster | AWS EKS |
| Cloud registry | GitHub Container Registry (GHCR) |
| App | Python (FastAPI) |
| Package manager | Helm |
| Traffic routing | Gateway API (Envoy Gateway) |
| Policy | Pod Security Standards + ValidatingAdmissionPolicy |
| IaC | Terraform |
| CI | GitHub Actions + Trivy |
| CD / GitOps | ArgoCD |
| Monitoring | Prometheus + Grafana |

---

## Folder Structure

not available yet.


---

## Installation

not available yet.

---

## Conventions

- Never use `:latest` image tag — always use a specific version (`:v1.0.0`)
- Never commit real Secret values — use `secret.yaml.example` as a template

---

## Roadmap

Detailed execution roadmap is in [ROADMAP.md](ROADMAP.md).
