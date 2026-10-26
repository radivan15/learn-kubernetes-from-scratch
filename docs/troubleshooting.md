# Troubleshooting

Issues hit while building this project, written up as symptom → cause → fix. Environment: Windows 11, Docker Desktop, k3d.

---

## `kubectl` fails with "connection refused" after `k3d cluster create`

**Symptom:**

```
E0914 ... couldn't get current server API group list: Get "https://host.docker.internal:PORT/api?timeout=32s":
dial tcp <LAN IP>:PORT: connectex: No connection could be made because the target machine actively refused it.
```

`k3d cluster list` and `docker ps -a` both show the cluster's containers as running/healthy — this is not a cluster startup failure.

**Cause:** k3d writes `host.docker.internal` as the API server address in kubeconfig. On this Windows + Docker Desktop setup, that hostname resolves to the WiFi LAN IP instead of a reachable loopback address, even though the k3d proxy container correctly exposes the API port on `127.0.0.1`. Confirmed with:

```powershell
Test-NetConnection -ComputerName 127.0.0.1 -Port <PORT>              # succeeds
Test-NetConnection -ComputerName host.docker.internal -Port <PORT>   # fails
```

**Fix:** point kubeconfig at `127.0.0.1` instead of `host.docker.internal`:

```powershell
kubectl config view --minify   # find the PORT and cluster name (e.g. k3d-dev-cluster)
kubectl config set-cluster k3d-<cluster-name> --server=https://127.0.0.1:<PORT>
```

This has to be repeated every time the cluster is recreated, since k3d rewrites kubeconfig on each `cluster create`.

**Prevention:** pin the API address in the k3d config file. k3d then writes `https://127.0.0.1:6550` into kubeconfig directly, so no manual fix is needed after recreating the cluster (verified with k3d v5.9.0):

```yaml
kubeAPI:
  hostIP: "127.0.0.1"
  hostPort: "6550"
```

---

## `Service` of type `LoadBalancer` stuck at `EXTERNAL-IP: <pending>`

**Symptom:** `kubectl get svc` shows `<pending>` indefinitely, and `kubectl get pods -A` shows a `svclb-<service-name>-xxxxx` Pod stuck `Pending` in `kube-system`.

**Cause:** k3s ships its own minimal load balancer implementation, ServiceLB (formerly called Klipper LB). It implements `LoadBalancer` by binding the Service's port directly as a `hostPort` on the Node. On a single-node cluster, every `LoadBalancer` Service competes for the same Node's ports — including k3s's default Traefik ingress controller, which claims ports `80`/`443` out of the box. `kubectl describe pod` on the stuck `svclb-*` Pod shows `FailedScheduling: ... node(s) didn't have free ports for the requested pod ports`.

Unlike a real cloud provider (e.g. AWS ELB), where each `LoadBalancer` Service gets its own dedicated external IP, ServiceLB reuses the Node IP for all of them — so port collisions here are a local-dev limitation, not something that occurs in production.

**Fix:** this project disables Traefik entirely, since traffic routing goes through Gateway API (Envoy Gateway) instead:

```powershell
k3d cluster delete dev-cluster
k3d cluster create dev-cluster --k3s-arg "--disable=traefik@server:0"
```

Recreating the cluster also resets kubeconfig, so the `host.docker.internal` fix above has to be applied again.

If you can't recreate the cluster, or hit the same conflict between two of your own `LoadBalancer` Services, use distinct ports per Service instead.
