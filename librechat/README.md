
## deployment
1. `k create namespace librechat`
1. fill in placeholders in secret-example.yaml and apply
   ```bash
   cp secret-example.yaml secret.yaml
   # edit secret.yaml
   k apply -f secret.yaml
   ```
   Use https://www.librechat.ai/toolkit/creds_generator
1. Use arm64 mongodb image
   `helm dependency build ./LibreChat/helm/librechat`
1. `helm install -n librechat librechat ./LibreChat/helm/librechat --values values.yaml`
