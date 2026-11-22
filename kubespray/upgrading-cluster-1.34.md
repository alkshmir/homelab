# Upgrading Cluster to Kubernetes v1.34.3 (Kubespray v2.30.0)

2026-11-22〜23 実施。Kubespray を `v2.30.0` へ更新し、Kubernetes を `v1.33.7` から `v1.34.3` へアップグレードした作業ログ。

- 前回のアップグレード記録: [upgrading-cluster-1.33.md](./upgrading-cluster-1.33.md)
- 事前実施した Longhorn アップグレード記録: [longhorn/upgrading-1.12.md](../longhorn/upgrading-1.12.md)

---

## 1. 対象構成とバージョン変更結果

| ノード名 | IP | ハードウェア / OS | 役割 |
| :--- | :--- | :--- | :--- |
| `node1-master` | `192.168.1.137` | Raspberry Pi 4 / Debian 12 (arm64) | `kube_control_plane`, `etcd`, `kube_node` |
| `node2-worker` | `192.168.1.118` | Raspberry Pi 5 / Debian 12 (arm64) | `kube_node` |
| `nucboxg3-1` | `192.168.1.99` | NucBox G3 / Debian 13 Trixie (amd64) | `kube_node` |

| コンポーネント | アップグレード前 | アップグレード後 | 備考 |
| :--- | :--- | :--- | :--- |
| Kubespray | `v2.29.1` | **`v2.30.0`** (`f4ccdb5e7`) | |
| Kubernetes | `v1.33.7` | **`v1.34.3`** | 全 3 ノード更新完了 |
| containerd | `2.1.5` | **`2.2.1`** | 全 3 ノード更新完了 |
| Cilium | `1.18.4` | **`1.18.6`** | `cilium_extra_values` により `k8sServiceHost` 明示・`cilium-envoy` 抑止済み |
| etcd | `3.5.25` | **`3.5.26`** | `etcd_deployment_type: host` |
| CoreDNS | `1.12.0` | **`1.12.1`** | |
| Longhorn | `v1.10.2` → `v1.12.2` | **`v1.12.2`** | 本作業前に先行アップグレード済み（K8s `1.33`/`1.34`/`1.35` 対応） |

---

## 2. 事前作業・準備

### 2.1 Longhorn 先行アップグレード & 事前確認

1. **Longhorn を `v1.12.2` へ先行アップグレード**
   - 将来の K8s `v1.35` も見据え、現行 `v1.33.7` 上で Longhorn を `v1.10.2` → `v1.11.3` → `v1.12.2` へ先行更新し、`node1-master` / `node2-worker` で `crictl rmi --prune` を実施（詳細: [longhorn/upgrading-1.12.md](../longhorn/upgrading-1.12.md)）。
2. **環境前提確認**
   - 全 3 ノードが **cgroup v2 (`cgroup2fs`)** 稼働済みであることを確認。
   - `node1-master` 上で `sudo /usr/local/bin/cilium version` が `cilium image (running): 1.18.4` を正常認識していることを確認。
3. **etcd スナップショット取得（`node1-master`）**
   ```bash
   sudo ETCDCTL_API=3 /usr/local/bin/etcdctl \
     --endpoints=https://127.0.0.1:2379 \
     --cacert=/etc/ssl/etcd/ssl/ca.pem \
     --cert=/etc/ssl/etcd/ssl/member-node1-master.pem \
     --key=/etc/ssl/etcd/ssl/member-node1-master-key.pem \
     snapshot save /tmp/etcd-snapshot-pre-1.34.db
   ```
   - `HASH: 1b34d02f` / `REVISION: 1087492037` / `TOTAL KEYS: 11344` / `109 MB` を取得しローカルへ `scp` 退避。
4. **Cilium Helm values 退避**
   - `helm get values cilium -n kube-system -a` をローカルに退避。

### 2.2 Kubespray `v2.30.0` チェックアウト & `venv` 更新

```bash
cd kubespray/kubespray
git checkout v2.30.0
./venv/bin/pip install -r requirements.txt
```

- [.gitmodules](../.gitmodules) の `branch` を `release-2.29` → `release-2.30` に更新。
- `venv` は `ansible==10.7.0` (`ansible-core 2.17.14`) 維持で `cryptography==46.0.3` / `jmespath==1.1.0` へ更新。

### 2.3 Inventory 更新（`v2.30.0` 変更点への対応）

`v2.29.1` → `v2.30.0` の `inventory/sample` 差分を反映しつつ、以下の環境カスタマイズを適用した：

- [k8s-cluster.yml](./inventory/group_vars/k8s_cluster/k8s-cluster.yml):
  - `kube_version: 1.34.3`
  - `kube_owner: root`、`kube_network_plugin: cilium` を維持
- [k8s-net-cilium.yml](./inventory/group_vars/k8s_cluster/k8s-net-cilium.yml):
  - `v2.30.0`（PR `#12624`）で `k8sServiceHost` が `kube_apiserver_global_endpoint` 由来に変更された。本クラスタは外部 LB なしのため未指定だと `localhost:6443` となり、`nucboxg3-1` の `/etc/hosts` の `::1 localhost` と `nginx-proxy`（IPv4 のみ待ち受け）の不整合リスクがあったため、`cilium_extra_values` で明示指定して回避した：
    ```yaml
    cilium_extra_values:
      k8sServiceHost: "192.168.1.137"
      k8sServicePort: "6443"
      envoy:
        enabled: false
    ```
- [containerd.yml](./inventory/group_vars/all/containerd.yml): `v2.30.0` のコメント差分（`options.Root`）を反映。

`kubespray/kubespray/inventory/mycluster` へ同期し、`ansible-inventory` および `--syntax-check` を通過させた。

---

## 3. Playbook 実行ログ

### 3.1 事前疎通確認（`playbooks/facts.yml`）

```bash
ansible-playbook -i inventory/mycluster/hosts.yaml \
  -u $USERNAME -b -v playbooks/facts.yml \
  --ask-become-pass | tee ../facts-1.34.log
```

```text
PLAY RECAP *********************************************************************
node1-master               : ok=31   changed=0    unreachable=0    failed=0    skipped=23   rescued=0    ignored=0
node2-worker               : ok=18   changed=0    unreachable=0    failed=0    skipped=18   rescued=0    ignored=0
nucboxg3-1                 : ok=18   changed=0    unreachable=0    failed=0    skipped=18   rescued=0    ignored=0
```

### 3.2 クラスタアップグレード（`upgrade-cluster.yml`）

前回の教訓を踏まえ、`upgrade_node_confirm` はデフォルト `false` のまま実行：

```bash
ansible-playbook -i inventory/mycluster/hosts.yaml \
  -u $USERNAME -b -v upgrade-cluster.yml \
  -e kube_version=1.34.3 \
  -e "serial=1" \
  -e "drain_fallback_enabled=true" \
  --ask-become-pass | tee ../upgrade-1.34.log
```

```text
PLAY RECAP *********************************************************************
node1-master               : ok=612  changed=84   unreachable=0    failed=0    skipped=1193 rescued=0    ignored=0
node2-worker               : ok=441  changed=39   unreachable=0    failed=0    skipped=665  rescued=0    ignored=0
nucboxg3-1                 : ok=434  changed=39   unreachable=0    failed=0    skipped=688  rescued=0    ignored=0
```

- 所要時間 **1 時間 24 分 13 秒**、**1 回で全ノード `failed=0` で完了**。
- **経過メモ（`nucboxg3-1` の drain 待機時間）**: Longhorn の `instance-manager` の PDB（`ALLOWED DISRUPTIONS: 0`）により通常 drain がブロックされたため、Kubespray デフォルト設定（`drain_timeout=360s` × 初回＋リトライ 3 回 ＝ 約 24 分半）の経過後に `drain_fallback_enabled=true` の `--disable-eviction` が自動発動して通過した。

---

## 4. 最終検証結果

### ノード・バージョン

```text
Server Version: v1.34.3

NAME           STATUS   ROLES           AGE    VERSION   INTERNAL-IP     OS-IMAGE                         CONTAINER-RUNTIME
node1-master   Ready    control-plane   672d   v1.34.3   192.168.1.137   Debian GNU/Linux 12 (bookworm)   containerd://2.2.1
node2-worker   Ready    <none>          672d   v1.34.3   192.168.1.118   Debian GNU/Linux 12 (bookworm)   containerd://2.2.1
nucboxg3-1     Ready    <none>          462d   v1.34.3   192.168.1.99    Debian GNU/Linux 13 (trixie)     containerd://2.2.1
```

### Helm リリース

```text
NAME     	NAMESPACE      	REVISION	STATUS  	CHART          	APP VERSION
cilium   	kube-system    	6       	deployed	cilium-1.18.6  	1.18.6
librechat	librechat      	6       	deployed	librechat-3.2.1	v0.8.9
longhorn 	longhorn-system	6       	deployed	longhorn-1.12.2	v1.12.2
metallb  	metallb-system 	1       	deployed	metallb-0.14.9 	v0.14.9
traefik  	traefik        	2       	deployed	traefik-42.4.1 	v3.7.19
zot      	zot            	2       	deployed	zot-0.1.135    	v2.1.23
```

### コンポーネント・ワークロード確認

- **Cilium**: 全 3 ノード `cilium` Pod `1/1 Running`、`cilium-operator` `2/2 Running`、`cilium-envoy` 未作成を確認（`k8sServiceHost: 192.168.1.137` / `envoy.enabled: false` とも正常機能）。
- **kube-system**: Control Plane・CoreDNS (2/2)・nodelocaldns (3/3)・kube-proxy (3/3)・nginx-proxy (2/2)・metrics-server・external-dns すべて `Running`。
- **Longhorn (`v1.12.2`)**: 全 6 ボリューム `attached / healthy`。全ノードの `drain` により旧 `v1.10.2` の `instance-manager` Pod 3 体も完全に整理され、全 3 ノードとも `v1.12.2` のみに統一された。
- **ワークロード**: `homelab-dns`, `librechat`, `metallb`, `traefik`, `wg-easy`, `zot` 正常稼働（`librechat-librechat` は `nucboxg3-1` drain 中の MongoDB デタッチに伴い一時リスタート後、MongoDB 復帰に伴い復旧）。

---

## 5. 次回（`v1.35.x` 等）への申し送り

- **drain 時間短縮**: Longhorn 環境では `-e "drain_fallback_enabled=true" -e "drain_retries=0" -e "drain_timeout=60s"` を併用すると、PDB ブロック時の待ち時間（約 24 分/ノード）を約 1 分へ短縮できる。
- **ローカル `kubectl`**: 現行クライアントが `v1.32.0` のためサーバ `v1.34.3` に対して Version Skew Policy (±1) の警告が出る。次作業までに `kubectl` も `v1.34.x` へ更新推奨。
- **`kube-proxy` IPVS モード**: `v1.35` で非推奨化（`v1.40` で削除予定）されるため、`v1.35` 以降のタイミングで `nftables` または Cilium `kubeProxyReplacement` への移行を検討する。
- **Longhorn**: `v1.12.2` は既に K8s `1.35` 対応済み（必要なら K8s `1.34+` 上で `v1.13.0` へ上げることも可能）。
