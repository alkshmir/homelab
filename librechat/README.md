
## Deployment

Ref: [LibreChat Helm Chart (chart-3.2.1)](https://github.com/LibreChat-AI/LibreChat/releases/tag/chart-3.2.1)

1. `kubectl create namespace librechat`
1. Fill in placeholders in `secret-example.yaml` and apply:
   ```bash
   cp secret-example.yaml secret.yaml
   # edit secret.yaml
   kubectl apply -f secret.yaml
   ```
   Use https://www.librechat.ai/toolkit/creds_generator
1. Install or upgrade via the official OCI Helm chart:
   ```bash
   helm upgrade --install librechat oci://ghcr.io/librechat-ai/librechat-chart/librechat \
     -n librechat --version 3.2.1 --values values.yaml
   ```
1. `kubectl apply -f httproute.yaml`

## `values.yaml` customizations

- `global.librechat.existingSecretName`: Reference `librechat-credentials-env` for `CREDS_KEY`, `CREDS_IV`, `JWT_SECRET`, `JWT_REFRESH_SECRET`, `MEILI_MASTER_KEY`, and provider API keys.
- `updateStrategy.type`: Set to `Recreate` because `imageVolume` uses a `ReadWriteOnce` Longhorn PVC.
- `ingress.enabled`: Disabled (exposed via Gateway API `HTTPRoute` in `httproute.yaml`).
- `mongodb`: Pinned to `amd64` nodes (`kubernetes.io/arch: amd64`), `updateStrategy.type: Recreate` (for RWO PVC lock safety), `image.tag: 7.0.15` (pulled from `bitnamilegacy/mongodb`), and `auth.enabled: false` to preserve compatibility with the existing unauthenticated PVC.
- `meilisearch`: Uses `longhorn` (`8Gi`) and `librechat-credentials-env` for `MEILI_MASTER_KEY`.
