# Roadmap — Learning Kubernetes

Phases 0–6 run entirely on a local k3d cluster (free). All paid AWS work is grouped into Phase 7, so by then every tool is already familiar and the EKS cluster can be destroyed after each session.

## Phase 0 — Setup & App `[LOCAL]`
> Reproducible local cluster + the app that will be deployed in every later phase

- [x] Install k3d, kubectl, Helm
- [x] k3d config file (`k3d cluster create --config`) — 1 server + 2 agents, Traefik disabled, API on `127.0.0.1`, pinned k3s version
- [x] Add the local registry to the k3d config (`dev-registry.localhost:5000`)
- [x] Verify: `kubectl cluster-info` & `kubectl get nodes` (3 nodes Ready)
- [x] Pick a sample app image (`traefik/whoami`, pinned tag) — the real app lives in a separate repo, this repo is infra only
- [x] Re-tag and push the image to the k3d registry (`docker pull` → `docker tag` → `docker push`) and confirm the cluster can pull it (first Pod `Running`, reached via `kubectl port-forward`)

## Phase 1 — Kubernetes Local: Core Concepts `[LOCAL]`
> Understand core Kubernetes with hands-on practice

- [ ] Pod, Node, Cluster — difference from plain Docker containers
- [ ] First YAML manifest: deploy a simple Pod (the sample app image) — `k8s/pod.yaml`, `apply` is idempotent
- [ ] kubectl debugging from day one — `logs`, `describe`, `exec`, `get events`
- [ ] Labels & Selectors — the glue that connects Deployment → Pod → Service
- [ ] Deployment & ReplicaSet 
- [ ] Rolling Update & Rollback — `kubectl rollout status`, `kubectl rollout undo`
- [ ] Service — ClusterIP, NodePort, LoadBalancer
- [ ] Namespace — isolate dev/staging environments
- [ ] Create `dev` (imperative) and `staging` (declarative YAML) namespaces
- [ ] Same manifest deployed to multiple namespaces with `-n`
- [ ] Cross-namespace DNS — `<svc>.<ns>.svc.cluster.local`
- [ ] Switch default namespace in kubectl context
- [ ] Delete a namespace — everything inside goes with it
- [ ] ConfigMap & Secret
- [ ] ConfigMap injected as environment variables (`envFrom`)
- [ ] ConfigMap as files (volume mount)
- [ ] Secret — base64 is encoding, not encryption
- [ ] Config changes — which need a Pod restart, which don't

## Phase 2 — Kubernetes Local: Intermediate `[LOCAL]`
> Resources, health checks, autoscaling, storage, scheduling, traffic routing

- [ ] Resource requests & limits — QoS classes (Guaranteed/Burstable/BestEffort), what `OOMKilled` means
- [ ] Startup, liveness & readiness probe — using the sample app's health endpoint
- [ ] Metrics Server (bundled with k3s) & `kubectl top`
- [ ] Horizontal Pod Autoscaler (HPA) — why it needs `requests` to work
- [ ] Persistent Volume & PVC
- [ ] StatefulSet — PostgreSQL with automatic PVC
- [ ] Job & CronJob — batch tasks, scheduled cleanup (the Kubernetes equivalent of Linux cron)
- [ ] DaemonSet — run one Pod per node (how logging/monitoring agents are deployed)
- [ ] Taints & Tolerations, Node Affinity/Anti-Affinity — control which node a Pod can (or can't) land on
- [ ] PodDisruptionBudget — keep the app up during `kubectl drain` on an agent node
- [ ] Helm as a consumer — `repo add`, `install`, `upgrade`, `rollback`, overriding values
- [ ] Gateway API — install Envoy Gateway via Helm, then GatewayClass → Gateway → HTTPRoute

## Phase 3 — Kubernetes Security Fundamentals `[LOCAL]`
> RBAC, network isolation, and policy enforcement — security foundations that apply regardless of cloud provider

- [ ] RBAC — Role, RoleBinding, ClusterRole, ClusterRoleBinding
- [ ] ServiceAccount — least privilege, bound (expiring) tokens instead of long-lived legacy tokens
- [ ] NetworkPolicy — default-deny, then explicit allow rules between namespaces (verify enforcement with k3s's built-in network policy controller)
- [ ] Pod Security Standards — `pod-security.kubernetes.io` Namespace labels, `restricted` profile (the official replacement for the removed PodSecurityPolicy)
- [ ] ValidatingAdmissionPolicy (built-in, CEL) — enforce the [Conventions](README.md#conventions) (no `:latest`, resource limits required) automatically
- [ ] (Bonus) Kyverno — same policies, compare with ValidatingAdmissionPolicy
- [ ] (Bonus) Cilium + Hubble — network flow visibility (requires recreating the cluster with Flannel disabled)

## Phase 4 — Monitoring & Custom Helm Chart `[LOCAL]`
> Package the app with Helm and monitor the cluster

- [ ] Build a custom Helm chart for the sample app (swap in the separate-repo app image later)
- [ ] Deploy app via `helm install` to the dev namespace
- [ ] Multi-environment: `values-dev.yaml` (1 replica, HPA off) + `values-prod.yaml` (3 replicas, HPA on)
- [ ] Deploy Prometheus + Grafana via Helm (`kube-prometheus-stack`)
- [ ] Scrape app metrics — ServiceMonitor (needs an app exposing `/metrics`, from the separate app repo)
- [ ] Build a simple Grafana dashboard

## Phase 5 — CI & GitOps `[LOCAL]`
> Automate build and deployment like a real company — without paying for a cluster

- [ ] (App repo) GitHub Actions — build & push image to GHCR on git push (tag = version / commit SHA)
- [ ] (App repo) Scan the image with Trivy in CI — fail the pipeline on critical vulnerabilities
- [ ] Validate manifests / Helm chart in CI (`helm lint`, kubeconform)
- [ ] ArgoCD on k3d — cluster auto-syncs from Git
- [ ] Multi-environment with ArgoCD (dev/staging/prod)

## Phase 6 — Infrastructure as Code (Terraform) `[LOCAL]`
> Learn Terraform for free before using it to build paid infrastructure

- [ ] Terraform basics — resource, variable, output, state
- [ ] Practice against k3d with the Kubernetes/Helm provider
- [ ] Terraform modules
- [ ] Remote state in S3 with native locking (`use_lockfile = true`) — DynamoDB locking is deprecated

## Phase 7 — AWS EKS via Terraform `[CLOUD — PAID]`
> Production-grade cluster in the cloud — `terraform destroy` at the end of every session

- [ ] EKS vs self-managed Kubernetes — what AWS manages vs what you manage (incl. EKS Auto Mode as a comparison)
- [ ] VPC + EKS cluster with Terraform — node type `t3.medium` (`t3.small` only fits ~11 Pods with the VPC CNI)
- [ ] IAM for workloads — EKS Pod Identity (recommended) vs IRSA (still common)
- [ ] AWS Load Balancer Controller — expose Envoy Gateway through an NLB, same HTTPRoute manifests as local
- [ ] Pull app images from GHCR
- [ ] EBS CSI Driver — persistent storage (default StorageClass: GP3)
- [ ] Cluster Autoscaler — concept only (scale nodes based on pending Pods)
- [ ] Karpenter — node autoscaling in practice
- [ ] Secrets Manager + External Secrets Operator
- [ ] Point ArgoCD at EKS — the same Git repo now deploys to the cloud
- [ ] Upgrade the cluster's Kubernetes minor version — node group rolling upgrade, understand standard vs extended support cost
- [ ] Velero — backup PV snapshots + cluster state to S3, then practice a restore
- [ ] (Bonus) Full stack: VPC + EKS + RDS via Terraform
