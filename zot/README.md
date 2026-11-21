## Deployment

Ref: [zot Kubernetes install guide (v2.1.23)](https://zotregistry.dev/v2.1.23/install-guides/install-guide-k8s/#step-1-locate-the-helm-charts-in-a-remote-repository)

1. Add Helm repo and install:
   ```bash
   helm repo add project-zot http://zotregistry.dev/helm-charts
   helm repo update project-zot
   helm install zot project-zot/zot -n zot --create-namespace --version 0.1.135 --values values.yaml
   ```
1. Access `http://zot.homelab.local/`

## `values.yaml` customizations

- `service.type`: Use `ClusterIP` (exposed via Ingress instead of `NodePort`).
- `ingress`: Expose via `nginx` ingress at `http://zot.homelab.local` with unlimited body size and 600s proxy timeouts.
- `persistence` / `pvc`: Enable PVC (`5Gi`) for `/var/lib/registry`.
- `mountConfig` / `configFiles`: Enable `search` and `ui` extensions and set log level to `info`.
