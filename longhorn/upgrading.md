# Upgrading Longhorn (`v1.7.2` → `v1.10.2`)

Kubernetes `v1.33` 対応のため、Longhorn を `v1.7.2` から `v1.10.2` へ段階アップグレードした作業記録（2026-11-22 実施）。

## 1. 事前整理

### 1.1 孤立ボリュームの削除

`node2-worker` がディスク容量不足で `DiskPressure`（`Schedulable: False`）になっていたため、PV が存在しない孤立 Volume 2 本（計 13 GiB）を削除した。

| Volume | サイズ | 状態 | 元 PVC / Namespace |
| :--- | :--- | :--- | :--- |
| `pvc-b698a2e6-63c1-448d-881c-e023444c7327` | 5 GiB | `detached / unknown` | `harbor/data-harbor-trivy-0` |
| `pvc-e2930b9a-e4d9-41b1-88c4-c60f5e889741` | 8 GiB | `attached / healthy` | `librechat/librechat-meilisearch`（現行は別 PV を使用） |

```bash
kubectl -n longhorn-system patch volume.longhorn.io pvc-e2930b9a-e4d9-41b1-88c4-c60f5e889741 --type=merge -p '{"spec":{"nodeID":""}}'
kubectl -n longhorn-system delete volume.longhorn.io pvc-b698a2e6-63c1-448d-881c-e023444c7327 pvc-e2930b9a-e4d9-41b1-88c4-c60f5e889741
```

削除後、`node2-worker` のディスクが `Schedulable: True` に回復した。

### 1.2 Helm リポジトリ追加・CRD 確認

```bash
helm repo add longhorn https://charts.longhorn.io
helm repo update longhorn

# 全 Longhorn CRD の storedVersions が v1beta2 であることを確認
kubectl get crd -l app.kubernetes.io/name=longhorn -o=jsonpath='{range .items[*]}{.metadata.name}{": "}{.status.storedVersions}{"\n"}{end}'
```

---

## 2. 段階アップグレード手順

Longhorn はマイナーバージョンの飛び越しが不可（`preUpgradeChecker` でブロックされる）のため、1 マイナーバージョンずつ以下を実施した：

`v1.7.2` -> `v1.8.2` -> `v1.9.2` -> `v1.10.2`

各ステップで実施した操作：

1. **Helm アップグレード**
   ```bash
   helm upgrade longhorn longhorn/longhorn \
     --namespace longhorn-system \
     --version <VERSION> \
     --values values.yaml \
     --timeout 15m
   ```
2. **ボリュームエンジンのライブアップグレード**
   - `concurrent-automatic-engine-upgrade-per-node-limit` を一時的に `1` へ変更し、稼働中の全 6 ボリュームを無停止で新エンジンイメージへ追従させた：
     ```bash
     kubectl -n longhorn-system patch setting.longhorn.io concurrent-automatic-engine-upgrade-per-node-limit --type=merge -p '{"value":"1"}'
     ```
3. **収束確認と旧エンジンイメージ削除**
   - 全コンポーネント（`longhorn-manager`, `longhorn-csi-plugin`, `csi-attacher` / `provisioner` / `resizer` / `snapshotter`, `longhorn-driver-deployer`, `longhorn-ui`）が `Running`、かつ全 6 ボリュームが新バージョン・`healthy` になったことを確認。
   - `REFCOUNT=0` になった旧 `engineimage.longhorn.io` を削除して次バージョンへ進んだ（`ei-51cc7b9c` `v1.7.2` → `ei-17ff8b24` `v1.8.2` → `ei-27581b25` `v1.9.2`）。
4. **設定復元（`v1.10.2` 完了後）**
   ```bash
   kubectl -n longhorn-system patch setting.longhorn.io concurrent-automatic-engine-upgrade-per-node-limit --type=merge -p '{"value":"0"}'
   ```

---

## 3. 作業中に発生した事象と対処

### 3.1 `longhorn-csi-plugin` 削除のタイムアウト（`v1.8.2` 移行時）

- **現象**: `longhorn-driver-deployer` が旧 `v1.7.2` の `DaemonSet/longhorn-csi-plugin` を `foregroundDeletion` で削除した直後にタイムアウトし `CrashLoopBackOff` になった。Pod 自体は削除済みだったが DaemonSet の `foregroundDeletion` finalizer が外れずに残っていた。
- **原因**: デプロイ負荷により `node1-master`（Raspberry Pi 4）上の `kube-controller-manager` が leader election タイムアウトを起こして約 5 分間の `CrashLoopBackOff` に入り、Garbage Collector が停止していたため。
- **対処**: `kube-controller-manager` の復帰（`1/1 Running`）を待って `longhorn-driver-deployer` Pod を再起動し、`v1.8.2` の `longhorn-csi-plugin` を正常に再作成させた。

### 3.2 `librechat-mongodb` ボリュームの一時的な `degraded`（`v1.9.2` 移行時）

- **現象**: 6 本中 5 本は `v1.9.2 / healthy` になったが、`librechat-mongodb`（`pvc-907dc612-b371-4089-a320-f9eee780b538`）のみ `node2-worker` 側レプリカが停止し `degraded` のまま `v1.8.2` に留まった。
- **原因**: コンテナイメージの pull により `node2-worker` の空き容量が一時的に 25% 閾値（約 15.6 GB）を下回り `DiskPressure` になったため再構築できなかった。調査の結果、過去の孤立レプリカディレクトリ（`orphan.longhorn.io`）が 37 件蓄積していた。
- **対処**: 孤立レプリカを削除して約 8 GB の空き容量を回復させた上で、失敗したレプリカを削除して再構築を促し、`healthy` 化および `v1.9.2` エンジンへの自動アップグレードを完了させた。
  ```bash
  kubectl -n longhorn-system delete orphan.longhorn.io --all
  kubectl -n longhorn-system delete replica.longhorn.io pvc-907dc612-b371-4089-a320-f9eee780b538-r-de310b67
  ```

---

## 4. 最終状態と補足

- **Helm release**: `longhorn-1.10.2`（`v1.10.2`, revision 4, `deployed`）
- **ボリューム**: 稼働中 6 本（`wg-easy`, `librechat` x3, `homelab-dns`, `zot`）すべて `attached / healthy`、`docker.io/longhornio/longhorn-engine:v1.10.2`（全ワークロード Pod は再起動なしで継続稼働）
- **ノードディスク**: `node2-worker`（空き約 32.9 GB）・`nucboxg3-1`（空き約 196.3 GB）ともに `Schedulable: True`
- **補足**:
  - ライブアップグレードの仕様上、稼働中ボリュームのエンジンプロセスを保持している旧 `v1.7.2` の `instance-manager` Pod が 2 体（`node2-worker` / `nucboxg3-1`）残存している。次回のボリューム再アタッチ（ノード `drain` 等）に伴い自動で `v1.10.2` 側へ移行・削除される。
