## Deployment

As official harbor repository does not support arm64 build (but [in discussion](https://github.com/goharbor/community/pull/262)), use bitnami build instead:

1. `helm install -n harbor harbor oci://registry-1.docker.io/bitnamicharts/harbor --values values.yaml --create-namespace`

1. Access `http://core.harbor.homelab.local/`
   See `values.yaml` for the default password for user `admin`.
