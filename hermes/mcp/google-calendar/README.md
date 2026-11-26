## Deployment

Ref: [nspady/google-calendar-mcp (`@cocal/google-calendar-mcp` v2.8.1)](https://github.com/nspady/google-calendar-mcp)

This deploys `google-calendar-mcp` in a dedicated `hermes-mcp` namespace so Google OAuth tokens stay completely isolated from the Hermes Kata VM. Only read/write tools are exposed via `ENABLED_TOOLS` (`delete-event` and `manage-accounts` are excluded).

### 1. Prepare Google Cloud OAuth credentials

1. Open Google Cloud Console and enable the **Google Calendar API**.
1. Configure the **OAuth consent screen** and add your Google account under Test users (note: in Testing mode, refresh tokens expire every 7 days unless you publish the app to Production or use a Service Account).
1. Create an **OAuth 2.0 Client ID** (Application type: **Desktop app**) and download the JSON (`gcp-oauth.keys.json`).

### 2. Deploy manifests

1. Create the `hermes-mcp` namespace:
   ```bash
   kubectl apply --server-side -f ../namespace.yaml
   ```
1. Create `secret.yaml` from `secret-example.yaml` with your `gcp-oauth.keys.json` and apply:
   ```bash
   cp secret-example.yaml secret.yaml
   # edit secret.yaml
   kubectl apply -f secret.yaml
   ```
1. Deploy PVC, NetworkPolicy, and the MCP server:
   ```bash
   kubectl apply -f pvc.yaml
   kubectl apply -f networkpolicy.yaml
   kubectl apply -f deployment.yaml
   kubectl -n hermes-mcp rollout status deployment/google-calendar-mcp --timeout=180s
   ```
1. Re-apply Hermes's `CiliumNetworkPolicy` so Hermes can reach `google-calendar-mcp` on port `3000`:
   ```bash
   kubectl apply -f ../../networkpolicy.yaml
   ```

### 3. One-time Google OAuth login

> Note: Do **not** use `http://localhost:3000/accounts` when `HOST=0.0.0.0` (its web UI sends `redirect_uri=http://0.0.0.0:3000/oauth2callback`, which Google rejects with `Error 400: invalid_request`). Use the built-in CLI `auth` flow on port `3500` instead, which uses `http://localhost:3500/oauth2callback`.

1. In terminal 1, forward port `3500` to the Pod:
   ```bash
   kubectl -n hermes-mcp port-forward deploy/google-calendar-mcp 3500:3500
   ```
1. In terminal 2, run the CLI auth command inside the Pod:
   ```bash
   kubectl -n hermes-mcp exec -it deploy/google-calendar-mcp -- \
     npx @cocal/google-calendar-mcp@2.8.1 auth
   ```
1. Open the printed Google authorization URL in your browser and approve access. Tokens are saved to `/tokens/tokens.json` on the `google-calendar-mcp-tokens` Longhorn PVC.

### 4. Register in Hermes

```bash
kubectl -n hermes exec -it hermes -- hermes mcp add google-calendar \
  --url http://google-calendar-mcp.hermes-mcp.svc.cluster.local:3000
kubectl -n hermes exec hermes -- hermes mcp test google-calendar
kubectl -n hermes delete pod hermes
```

## Security design

- **Token isolation**: OAuth client secret and `tokens.json` live only in `hermes-mcp`, never mounted into the `hermes` Kata VM.
- **Delete disabled server-side**: `ENABLED_TOOLS` exposes only:
  - Read: `list-calendars`, `list-events`, `get-event`, `search-events`, `get-freebusy`, `get-current-time`, `list-colors`
  - Write (no delete): `create-event`, `create-events`, `update-event`, `respond-to-event`
  - Excluded: `delete-event` (`manage-accounts` is always registered by the server; disable it in Hermes with `hermes mcp configure google-calendar` if desired)
- **Hardened container**: `pod-security.kubernetes.io/enforce: restricted`, non-root UID `1000`, `readOnlyRootFilesystem: true`, all capabilities dropped.
- **Network isolation**: `google-calendar-mcp` accepts TCP `3000` only from the `hermes` namespace and can only make outbound DNS + HTTPS (`443`) calls to the public internet.
