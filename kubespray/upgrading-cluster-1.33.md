# Upgrading Cluster to Kubernetes v1.33.7 (Kubespray v2.29.1)

2026-11-22 実施。Kubespray を `v2.29.1` へ更新し、Kubernetes を `v1.32.6` から `v1.33.7` へアップグレードした作業ログ。

- 前回のアップグレード記録: [upgradeing-cluster.md](./upgradeing-cluster.md)
- 事前実施した Longhorn アップグレード記録: [longhorn/upgrading.md](../longhorn/upgrading.md)

---

## 1. 対象構成とバージョン変更結果

| ノード名 | IP | ハードウェア / OS | 役割 |
| :--- | :--- | :--- | :--- |
| `node1-master` | `192.168.1.137` | Raspberry Pi 4 / Debian 12 (arm64) | `kube_control_plane`, `etcd`, `kube_node` |
| `node2-worker` | `192.168.1.118` | Raspberry Pi 5 / Debian 12 (arm64) | `kube_node` |
| `nucboxg3-1` | `192.168.1.99` | NucBox G3 / Debian 13 Trixie (amd64) | `kube_node` |

| コンポーネント | アップグレード前 | アップグレード後 | 備考 |
| :--- | :--- | :--- | :--- |
| Kubespray | `155c1c1` | **`v2.29.1`** (`0c6a29553`) | Debian 13 (Trixie) 正式サポート |
| Kubernetes | `v1.32.6` | **`v1.33.7`** | 全 3 ノード更新完了 |
| containerd | `2.1.3` | **`2.1.5`** | 全 3 ノード更新完了 |
| Cilium | `1.17.3` | **`1.18.4`** | `cilium_extra_values` により `cilium-envoy` 抑止済み |
| etcd | `3.5.21` | **`3.5.25`** | `etcd_deployment_type: host` |
| Longhorn | `v1.7.2` → `v1.10.2` | **`v1.10.2`** | 本作業前に先行アップグレード済み |

---

## 2. 事前作業・準備

### 2.1 バックアップ・事前確認

1. **Longhorn 先行アップグレード & 不要イメージ整理**
   - Kubernetes `v1.33` 対応のため、事前に Longhorn を `v1.7.2` → `v1.8.2` → `v1.9.2` → `v1.10.2` へアップグレードし、`node1-master` / `node2-worker` で `crictl rmi --prune` を実施（詳細: [longhorn/upgrading.md](../longhorn/upgrading.md)）。
2. **etcd スナップショット取得（`node1-master`）**
   ```bash
   sudo ETCDCTL_API=3 /usr/local/bin/etcdctl \
     --endpoints=https://127.0.0.1:2379 \
     --cacert=/etc/ssl/etcd/ssl/ca.pem \
     --cert=/etc/ssl/etcd/ssl/member-node1-master.pem \
     --key=/etc/ssl/etcd/ssl/member-node1-master-key.pem \
     snapshot save /tmp/etcd-snapshot-pre-1.33.db
   ```
   - `HASH: 468c9c6d` / `REVISION: 1087113564` / `TOTAL KEYS: 16596` / `107 MB` を取得しローカルへ `scp` 退避。
3. **Cilium 現状確認**
   - `helm get values cilium -n kube-system -a` をローカルに退避。
   - `node1-master` で `KUBECONFIG=/etc/kubernetes/admin.conf /usr/local/bin/cilium version` により `cilium image (running): 1.17.3` を確認。
   - 既に Helm 管理済みのため、前回使用した `cilium_remove_old_resources=true` は今回指定しない方針とした。

### 2.2 Kubespray `v2.29.1` チェックアウト & `venv` 再構築

```bash
cd kubespray/kubespray
git fetch --tags origin
git checkout v2.29.1
```

- [.gitmodules](../.gitmodules) の `branch` を `release-2.26` → `release-2.29` に更新。
- `roles/kubespray_defaults/vars/main/checksums.yml` に `1.33.7`（`arm64` / `amd64`）が存在することを確認。
- Python 3.12.1 で `venv` を再構築し、`requirements.txt`（`ansible==10.7.0` / `ansible-core 2.17.14`）をインストール。

### 2.3 Inventory 最新化

[inventory/](./inventory/) の `group_vars` を `v2.29.1` の `kubespray/inventory/sample/group_vars` ベースに刷新（廃止された `k8s-net-weave.yml` / 旧 `etcd.yml` を削除、`kube_control_plane.yml` を追加）し、以下の環境固有カスタマイズのみを適用：

- [k8s-cluster.yml](./inventory/group_vars/k8s_cluster/k8s-cluster.yml):
  - `kube_version: 1.33.7`
  - `kube_owner: root`（`#12276` 対策）
  - `kube_network_plugin: cilium`
- [k8s-net-cilium.yml](./inventory/group_vars/k8s_cluster/k8s-net-cilium.yml):
  - Raspberry Pi 4（`node1-master`）での `cilium-envoy` TCMalloc クラッシュ防止のため `cilium_extra_values` を追加：
    ```yaml
    cilium_extra_values:
      envoy:
        enabled: false
    ```
- [addons.yml](./inventory/group_vars/k8s_cluster/addons.yml):
  - `metrics_server_enabled: true`

その後 `kubespray/kubespray/inventory/mycluster` へ同期し、`ansible-inventory` および `--syntax-check` を通過させた。

---

## 3. Playbook 実行ログと発生事象

### 3.1 事前疎通確認（`playbooks/facts.yml`）

```bash
ansible-playbook -i inventory/mycluster/hosts.yaml \
  -u $USERNAME -b -v playbooks/facts.yml \
  --ask-become-pass | tee ../facts-1122.log
```

```text
PLAY RECAP *********************************************************************
node1-master               : ok=39   changed=0    unreachable=0    failed=0    skipped=22   rescued=0    ignored=0
node2-worker               : ok=26   changed=0    unreachable=0    failed=0    skipped=17   rescued=0    ignored=0
nucboxg3-1                 : ok=26   changed=0    unreachable=0    failed=0    skipped=17   rescued=0    ignored=0
```

### 3.2 1 回目実行（`upgrade-cluster.yml`）

```bash
ansible-playbook -i inventory/mycluster/hosts.yaml \
  -u $USERNAME -b -v upgrade-cluster.yml \
  -e kube_version=1.33.7 \
  -e "serial=1" \
  -e "drain_fallback_enabled=true" \
  -e "upgrade_node_confirm=true" \
  --ask-become-pass | tee ../upgrade-1122.log
```

```text
PLAY RECAP *********************************************************************
node1-master               : ok=586  changed=87   unreachable=0    failed=0    skipped=963  rescued=0    ignored=1
node2-worker               : ok=303  changed=24   unreachable=0    failed=1    skipped=258  rescued=0    ignored=0
nucboxg3-1                 : ok=298  changed=24   unreachable=0    failed=0    skipped=255  rescued=0    ignored=0
```

**結果と発生事象：**
- `etcd` (`3.5.25`)、`node1-master` (`v1.33.7`, `containerd://2.1.5`)、および `Cilium` (`1.18.4`, `cilium-envoy` 出現なし) は一発で成功。
- **事象 1（`upgrade_node_confirm=true` + `tee` の出力バッファリング）**: `PLAY [Handle upgrades to control plane...]` 冒頭の Enter 待ちプロンプトが `tee` へのパイプにより出力されず、約 25 分間止まったように見えた。次回からはデフォルト（`upgrade_node_confirm: false`）のまま実行する。
- **事象 2（`node2-worker` の電源断 → `/tmp/releases/` 消失）**: 待機中に `node2-worker` の電源が落ちており powercycle して復帰させた結果、序盤にダウンロード済みだった `/tmp/releases/cni-plugins-linux-arm64-1.8.0.tgz` が再起動で消え、`TASK [network_plugin/cni : CNI | Copy cni plugins]` で失敗した。

### 3.3 2 回目実行（リトライで完了）

`-e "upgrade_node_confirm=true"` を外してそのまま再実行：

```bash
ansible-playbook -i inventory/mycluster/hosts.yaml \
  -u $USERNAME -b -v upgrade-cluster.yml \
  -e kube_version=1.33.7 \
  -e "serial=1" \
  -e "drain_fallback_enabled=true" \
  --ask-become-pass | tee ../upgrade-1122-2.log
```

```text
PLAY RECAP *********************************************************************
node1-master               : ok=599  changed=40   unreachable=0    failed=0    skipped=1188 rescued=0    ignored=0
node2-worker               : ok=442  changed=38   unreachable=0    failed=0    skipped=665  rescued=0    ignored=0
nucboxg3-1                 : ok=435  changed=27   unreachable=0    failed=0    skipped=688  rescued=0    ignored=0
```

所要時間約 52 分（うち `Drain node` 合計約 25 分）で全ノード `failed=0` で完了。

---

## 4. 最終検証結果

### ノード・バージョン

```text
Server Version: v1.33.7

NAME           STATUS   ROLES           AGE    VERSION   INTERNAL-IP     OS-IMAGE                         CONTAINER-RUNTIME
node1-master   Ready    control-plane   671d   v1.33.7   192.168.1.137   Debian GNU/Linux 12 (bookworm)   containerd://2.1.5
node2-worker   Ready    <none>          671d   v1.33.7   192.168.1.118   Debian GNU/Linux 12 (bookworm)   containerd://2.1.5
nucboxg3-1     Ready    <none>          462d   v1.33.7   192.168.1.99    Debian GNU/Linux 13 (trixie)     containerd://2.1.5
```

### Helm リリース

```text
NAME     	NAMESPACE      	REVISION	STATUS  	CHART          	APP VERSION
cilium   	kube-system    	5       	deployed	cilium-1.18.4  	1.18.4
librechat	librechat      	6       	deployed	librechat-3.2.1	v0.8.9
longhorn 	longhorn-system	4       	deployed	longhorn-1.10.2	v1.10.2
metallb  	metallb-system 	1       	deployed	metallb-0.14.9 	v0.14.9
traefik  	traefik        	2       	deployed	traefik-42.4.1 	v3.7.19
zot      	zot            	2       	deployed	zot-0.1.135    	v2.1.23
```

### コンポーネント・ワークロード確認

- **Cilium**: 全 3 ノード `cilium` Pod `1/1 Running`、`cilium-operator` `2/2 Running`、`cilium-envoy` DaemonSet 未作成を確認（`cilium_extra_values` が期待通り機能）。
- **kube-system**: `kube-apiserver`, `kube-controller-manager`, `kube-scheduler`, `coredns` (2/2), `nodelocaldns` (3/3), `kube-proxy` (3/3), `nginx-proxy` (2/2), `metrics-server`, `dns-autoscaler`, `external-dns` すべて `Running`。
- **Longhorn**: 全 6 ボリューム `attached / healthy`。worker の `drain` に伴うボリューム再アタッチにより、残存していた旧 `v1.7.2` の `instance-manager` Pod も完全に整理され、全 3 ノードとも `v1.10.2` のみになった。
- **ワークロード**: `homelab-dns`, `librechat`, `metallb`, `traefik`, `wg-easy`, `zot` を含め全 Namespace の Pod が `Running` / `Completed`。

---

## 5. 次回アップグレードへの申し送り

- `tee` にパイプする際は `-e upgrade_node_confirm=true` を付けない（プロンプトがバッファリングされて止まったように見える）。
- `k8s-net-cilium.yml` の `cilium_extra_values: { envoy: { enabled: false } }` と `k8s-cluster.yml` の `kube_owner: root` は維持する。
- Longhorn が動いている環境では `-e "serial=1" -e "drain_fallback_enabled=true"` を継続使用する。
- 実行前に各ノードの不要イメージ整理（`sudo /usr/local/bin/crictl --timeout 120s rmi --prune`）および `node1-master` での etcd スナップショット取得を行う。
