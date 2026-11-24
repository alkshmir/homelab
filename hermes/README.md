## Deployment

Prerequisites:
- [Kata Containers](../kata) installed on `nucboxg3-1` (`runtimeClassName: kata` or `kata-clh-runtime-rs`)
- [Agent Sandbox](../agent-sandbox) CRD installed (`agents.x-k8s.io/v1beta1`)

1. Create namespace, default ServiceAccount (`automountServiceAccountToken: false`), resource limits, network isolation, and persistent volume:
   ```bash
   kubectl apply -f namespace.yaml
   kubectl apply -f resourcequota.yaml
   kubectl apply -f networkpolicy.yaml
   kubectl apply -f pvc.yaml
   ```

1. Fill in credentials from `secret-example.yaml` and apply:
   ```bash
   cp secret-example.yaml secret.yaml
   # edit secret.yaml
   kubectl apply -f secret.yaml
   ```

1. (Optional) Expose via Traefik Gateway API (`http://hermes.homelab.local`):
   ```bash
   kubectl apply -f httproute.yaml
   ```

1. Deploy Hermes as a Kata-backed `Sandbox`:
   ```bash
   kubectl apply -f sandbox.yaml
   kubectl -n hermes wait --for=condition=Ready sandbox/hermes --timeout=180s
   ```

1. Run initial setup / model selection (select Gemini) or chat inside the sandbox:
   ```bash
   kubectl -n hermes exec -it hermes -- hermes model
   kubectl -n hermes exec -it hermes -- hermes chat
   ```

   To pause/resume the Kata VM while keeping `hermes-data`:
   ```bash
   kubectl -n hermes patch sandbox hermes --type=merge -p '{"spec":{"operatingMode":"Suspended"}}'
   kubectl -n hermes patch sandbox hermes --type=merge -p '{"spec":{"operatingMode":"Running"}}'
   ```

## Manifests

- `namespace.yaml`: Creates `hermes` namespace with Pod Security Standards (`enforce: baseline`) and disables ServiceAccount token auto-mounting on `default`.
- `resourcequota.yaml`: Caps total CPU/memory (`3` CPU / `3Gi` RAM) and sets default container limits (`1` CPU / `2Gi` RAM) so workloads cannot exhaust `nucboxg3-1`.
- `networkpolicy.yaml`: `CiliumNetworkPolicy` applied to all pods in `hermes`:
  - **Ingress**: Allows traffic only from `traefik` pods (`traefik` namespace).
  - **Egress**: Allows DNS (`169.254.25.10` NodeLocal DNSCache & CoreDNS) and public internet only; blocks home LAN (`192.168.0.0/16`), cluster Pod/Service CIDRs (`10.0.0.0/8`, `fc00::/7`), and link-local ranges.
- `pvc.yaml`: `5Gi` Longhorn `ReadWriteOnce` PVC (`hermes-data`).
- `secret-example.yaml`: Template for `hermes-credentials` (`secret.yaml` is gitignored).
- `sandbox.yaml`: Kata-backed `Sandbox` (`agents.x-k8s.io/v1beta1`, `kata-clh-runtime-rs`) running `nousresearch/hermes-agent:v0.23.4` (`gateway run`).
- `httproute.yaml`: Gateway API `HTTPRoute` for `hermes.homelab.local` via `traefik-gateway`.

## Verify isolation

After applying `namespace.yaml`, `resourcequota.yaml`, and `networkpolicy.yaml`, you can test Kata + Cilium isolation with a temporary `Sandbox`:

```bash
kubectl -n hermes run net-test --rm -i --restart=Never \
  --image=busybox \
  --overrides='{"spec":{"runtimeClassName":"kata-clh-runtime-rs"}}' \
  -- sh -c '
    echo "--- Guest kernel ---"
    uname -a
    echo "--- Public internet (should succeed) ---"
    wget -qO- --timeout=5 https://example.com | head -c 40; echo
    echo "--- Home LAN (should fail/timeout) ---"
    wget -qO- --timeout=3 http://192.168.1.99:10250 || echo "BLOCKED"
    echo "--- Cluster internal API (should fail/timeout) ---"
    wget -qO- --timeout=3 https://10.233.0.1:443 || echo "BLOCKED"
  '
```
