## Deployment

Ref: [Traefik Kubernetes Gateway API](https://doc.traefik.io/traefik/reference/install-configuration/providers/kubernetes/kubernetes-gateway/)

1. Install Gateway API CRDs (v1.6.3, standard channel):
   ```bash
   kubectl apply --server-side -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.6.3/standard-install.yaml
   ```
1. Add Helm repo and install Traefik:
   ```bash
   helm repo add traefik https://traefik.github.io/charts
   helm repo update traefik
   helm upgrade --install traefik traefik/traefik -n traefik --create-namespace --version 42.4.1 --values values.yaml
   ```

## `values.yaml` customizations

- `ports.web.transport.respondingTimeouts.readTimeout`: Set to `"0"` (unlimited) so large uploads (e.g. Zot image push) and long requests are not cut off at the entrypoint default (`60s`).
- `ingressClass.enabled` / `providers.kubernetesIngress.enabled`: Disabled so Traefik runs alongside `ingress-nginx` without conflict.
- `providers.kubernetesGateway.enabled`: Enable the Kubernetes Gateway API provider (creates default `GatewayClass` `traefik` and `Gateway` `traefik-gateway`).
- `gateway.listeners.web.namespacePolicy.from`: Set to `All` so `HTTPRoute` resources in other namespaces can attach to `traefik-gateway`.
