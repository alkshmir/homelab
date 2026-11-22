# Upgrading Longhorn (`v1.10.2` → `v1.12.2`)

Kubernetes `v1.34` および将来の `v1.35` へのアップグレードに先立ち、現行 Kubernetes `v1.33.7` 上で Longhorn を `v1.10.2` から `v1.12.2` へ段階アップグレードした作業記録（2026-11-22 実施）。
前回の `v1.7.2` → `v1.10.2` の作業記録は [upgrading.md](./upgrading.md) を参照。

---

## 1. 事前確認

- **対応バージョンの確認**: `charts.longhorn.io` の `Chart.yaml` を確認したところ、`v1.11.3`・`v1.12.2` はいずれも `kubeVersion: >=1.25.0-0` であり現行 K8s `v1.33.7` 上でそのまま適用可能（`v1.12.2` は K8s `1.33` / `1.34` / `1.35` に対応。`v1.13.0` は `kubeVersion: >=1.34.0-0` のため K8s `v1.34` 到達後に適用可能）。
- **[values.yaml](./values.yaml) の検証**: 前回差分整理済みの `values.yaml` で `1.11.3` / `1.12.2` とも `helm template` がエラーなく通過することを確認（変更不要）。
- 事前バックアップとして `settings.longhorn.io` と `volumes.longhorn.io` をローカルへエクスポート。

---

## 2. 段階アップグレード手順

Longhorn のマイナーバージョン制約に従い、1 マイナーバージョンずつ以下を実施した：

`v1.10.2` → **`v1.11.3`** → **`v1.12.2`**

1. **Helm リポジトリ更新 & ステップ 1（`v1.10.2` → `v1.11.3`）**
   ```bash
   helm repo update longhorn

   helm upgrade longhorn longhorn/longhorn \
     --namespace longhorn-system \
     --version 1.11.3 \
     --values values.yaml \
     --timeout 15m

   # ボリュームエンジンの自動ライブアップグレードを有効化
   kubectl -n longhorn-system patch setting.longhorn.io concurrent-automatic-engine-upgrade-per-node-limit --type=merge -p '{"value":"1"}'
   ```
   - `longhorn-manager` (3/3)・`longhorn-csi-plugin` (3/3)・全 6 ボリュームが `v1.11.3 / healthy` になったことを確認。
   - `REFCOUNT=0` になった旧 `v1.10.2` エンジンイメージを削除：
     ```bash
     kubectl -n longhorn-system delete engineimage.longhorn.io ei-843accdd
     ```

2. **ステップ 2（`v1.11.3` → `v1.12.2`）**
   ```bash
   helm upgrade longhorn longhorn/longhorn \
     --namespace longhorn-system \
     --version 1.12.2 \
     --values values.yaml \
     --timeout 15m
   ```
   - `longhorn-manager` (3/3)・`longhorn-csi-plugin` (3/3)・`csi-*` (各 3/3)・全 6 ボリュームが `v1.12.2 / healthy` になったことを確認。
   - `REFCOUNT=0` の旧 `v1.11.3` エンジンイメージを削除し、エンジン自動更新設定を元の `0` へ復元：
     ```bash
     kubectl -n longhorn-system delete engineimage.longhorn.io ei-5ee94262
     kubectl -n longhorn-system patch setting.longhorn.io concurrent-automatic-engine-upgrade-per-node-limit --type=merge -p '{"value":"0"}'
     ```

3. **不要コンテナイメージの整理**
   - `node1-master` / `node2-worker` で `sudo /usr/local/bin/crictl --timeout 120s rmi --prune` を実行。

---

## 3. 作業中に発生した事象と対処

- **`node1-master`（RPi4）負荷による `kube-controller-manager` の一時的な `CrashLoopBackOff`**
  - 前回（`v1.7.2` → `v1.10.2`）と同様、`v1.11.3` / `v1.12.2` ロールアウト時の負荷で `kube-controller-manager` が一時的に leader election タイムアウトを起こし、`longhorn-driver-deployer` の `longhorn-csi-plugin` 再作成が待機状態になった。
  - `kube-controller-manager` の復帰後に `longhorn-driver-deployer` Pod を再起動（`kubectl -n longhorn-system delete pod -l app=longhorn-driver-deployer`）して速やかに収束させた。

---

## 4. 最終状態と補足

- **Helm release**: `longhorn-1.12.2`（`v1.12.2`, revision 6, `deployed`）
- **コンポーネント**: `longhorn-manager` (3/3)・`longhorn-csi-plugin` (3/3)・`csi-attacher` / `provisioner` / `resizer` / `snapshotter` (各 3/3)・`longhorn-driver-deployer` (1/1)・`longhorn-ui` (2/2)・`engine-image-ei-5830480e` (`v1.12.2`, 3/3) すべて `Running`
- **ボリューム**: 稼働中 6 本（`wg-easy`, `librechat` x3, `homelab-dns`, `zot`）すべて `attached / healthy`、`docker.io/longhornio/longhorn-engine:v1.12.2`（ワークロード無停止）
- **孤立リソース**: `orphans.longhorn.io` 0 件
- **ノードディスク（`crictl rmi --prune` 後）**:
  - `node1-master`: `df` 空き 34 GB (使用率 40%) / Longhorn 認識空き 34.0 GB（`Schedulable: True`）
  - `node2-worker`: `df` 空き 34 GB (使用率 40%) / Longhorn 認識空き 32.7 GB（`Schedulable: True`）
  - `nucboxg3-1`: Longhorn 認識空き 203.1 GB（`Schedulable: True`）
- **補足**:
  - ライブアップグレードの仕様上、稼働中ボリュームのエンジンプロセスを保持している旧 `v1.10.2` の `instance-manager` Pod が 3 体残存している。次回の Kubernetes `v1.34.3` アップグレード時のノード `drain`（ボリューム再アタッチ）に伴い自動で `v1.12.2` 側へ移行・削除される。
