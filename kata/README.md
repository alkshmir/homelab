## Deployment

Ref: [Kata Containers `kata-deploy` Helm chart](https://github.com/kata-containers/kata-containers/tree/main/tools/packaging/kata-deploy/charts/kata-deploy)

1. Install Kata Containers via the official OCI Helm chart (targets `amd64` nodes only via `values.yaml`):
   ```bash
   helm upgrade --install kata-deploy oci://ghcr.io/kata-containers/kata-deploy-charts/kata-deploy \
     --namespace kata-system --create-namespace \
     --version 4.4.0 \
     --values values.yaml \
     --timeout 10m
   ```

   > Note: `kata-deploy` runs in `job` mode by default (no always-on DaemonSet) and restarts `containerd` on the target node (`nucboxg3-1`) during installation, which briefly interrupts pods on that node.

1. Verify `RuntimeClass` and node label:
   ```bash
   kubectl get runtimeclass
   kubectl get nodes -l katacontainers.io/kata-runtime=true
   ```

1. Quick smoke test (check that the pod runs on a Kata guest kernel rather than the host kernel):
   ```bash
   kubectl run kata-test --rm -it --restart=Never \
     --image=busybox \
     --overrides='{"spec":{"runtimeClassName":"kata"}}' \
     -- uname -a
   ```

## `values.yaml` customizations

- `nodeSelector`: Restrict installation to `kubernetes.io/arch: amd64` (`nucboxg3-1`), leaving Raspberry Pi (`arm64`) nodes untouched.
- `shims`: Set `disableAll: true` and enable only `qemu-runtime-rs` (Kata 4.x default, broad compatibility) and `clh-runtime-rs` (Cloud Hypervisor, lighter memory footprint).
- `runtimeClasses.createDefault`: Enabled so pods can use `runtimeClassName: kata` (maps to `qemu-runtime-rs`), in addition to `kata-qemu-runtime-rs` and `kata-clh-runtime-rs`.

## Uninstall

```bash
helm uninstall kata-deploy --namespace kata-system
```
