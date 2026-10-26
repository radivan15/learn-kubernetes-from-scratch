## Debugging

Real issues hit along the way, each traced from symptom to root cause. Full write-ups in [troubleshooting.md](troubleshooting.md).

- **`kubectl` "connection refused" on a healthy cluster** — kubeconfig pointed at `host.docker.internal`, which resolved to the LAN IP instead of loopback. Isolated with `Test-NetConnection` against both addresses, fixed by pointing kubeconfig at `127.0.0.1`.
- **`ImagePullBackOff` pulling from the local registry** — the Pod used `localhost:5000/...`, but inside a k3d node `localhost` is the node itself. Read from `kubectl describe pod` Events (`dial tcp [::1]:5000: connection refused`); fixed by using the registry's in-cluster name `dev-registry.localhost:5000`.
- **`LoadBalancer` Service stuck at `<pending>`** — k3s's ServiceLB binds `hostPort`s on the Node, and Traefik already held ports 80/443. Found via `FailedScheduling` events on the `svclb-*` Pod; fixed by disabling Traefik in favor of Gateway API. A local-dev limitation that doesn't exist on cloud load balancers.
