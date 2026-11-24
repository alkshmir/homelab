## Deployment

Ref: [kubernetes-sigs/agent-sandbox](https://github.com/kubernetes-sigs/agent-sandbox)

> Note: The official Helm chart is source-only (`./helm/`); the official release ships single-file manifests (`sandbox-with-extensions.yaml` includes Core `Sandbox` + Extensions `SandboxTemplate` / `SandboxClaim` / `SandboxWarmPool`).

1. Install Agent Sandbox (Core + Extensions, `v1.0.12`):
   ```bash
   kubectl apply --server-side -f https://github.com/kubernetes-sigs/agent-sandbox/releases/download/v1.0.12/sandbox-with-extensions.yaml
   ```

1. Verify CRDs and controller:
   ```bash
   kubectl get crd | grep agents.x-k8s.io
   kubectl get pods -n agent-sandbox-system
   ```

1. Quick smoke test (create a Kata-backed `Sandbox` and verify guest kernel):
   ```bash
   kubectl apply -f - <<'EOF'
   apiVersion: agents.x-k8s.io/v1beta1
   kind: Sandbox
   metadata:
     name: kata-sandbox-test
   spec:
     podTemplate:
       spec:
         runtimeClassName: kata
         automountServiceAccountToken: false
         containers:
           - name: sandbox
             image: busybox
             command: ["sleep", "300"]
   EOF

   kubectl wait --for=condition=Ready sandbox/kata-sandbox-test --timeout=60s
   kubectl exec kata-sandbox-test -- uname -a
   kubectl delete sandbox kata-sandbox-test
   ```

## Uninstall

Check that no `Sandbox` resources are in use first (deleting CRDs cascade-deletes all custom resources):

```bash
kubectl get sandboxes -A
kubectl delete -f https://github.com/kubernetes-sigs/agent-sandbox/releases/download/v1.0.12/sandbox-with-extensions.yaml
```
