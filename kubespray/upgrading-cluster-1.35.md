# Upgrading Cluster to Kubernetes v1.35.4 (Kubespray v2.31.0)

2026-11-23 実施。Kubespray を `v2.31.0` へ更新し、Kubernetes を `v1.34.3` から `v1.35.4` へアップグレードした作業ログ。

- 前回のアップグレード記録: [upgrading-cluster-1.34.md](./upgrading-cluster-1.34.md)
- 事前実施済み Longhorn アップグレード記録: [longhorn/upgrading-1.12.md](../longhorn/upgrading-1.12.md)

---

## 1. 対象構成とバージョン変更結果

| ノード名 | IP | ハードウェア / OS | 役割 |
| :--- | :--- | :--- | :--- |
| `node1-master` | `192.168.1.137` | Raspberry Pi 4 / Debian 12 (arm64) | `kube_control_plane`, `etcd`, `kube_node` |
| `node2-worker` | `192.168.1.118` | Raspberry Pi 5 / Debian 12 (arm64) | `kube_node` |
| `nucboxg3-1` | `192.168.1.99` | NucBox G3 / Debian 13 Trixie (amd64) | `kube_node` |

| コンポーネント | アップグレード前 | アップグレード後 | 備考 |
| :--- | :--- | :--- | :--- |
| Kubespray | `v2.30.0` | **`v2.31.0`** (`1c9add489`) | `ansible==11.13.0` (`ansible-core 2.18.20`) |
| Kubernetes | `v1.34.3` | **`v1.35.4`** | 全 3 ノード更新完了（cgroup v2 稼働済み） |
| etcd | `3.5.26` | **`3.6.10`** | v2 ストア自動クリーンアップ経由でマイナー更新完了 |
| Cilium | `1.18.6` | **`1.19.3`** | `cilium_extra_values` 維持、`cilium-envoy` 抑止済み |
| containerd | `2.2.1` | **`2.2.3`** | 全 3 ノード更新完了 |
| runc | `1.3.4` | **`1.4.2`** | |
| CoreDNS | `1.12.1` | **`1.12.4`** | |
| Longhorn | `v1.12.2` | **`v1.12.2`** | K8s `1.35` 公式対応済みのため変更なし |

---

## 2. 事前作業・準備

### 2.1 バックアップ・事前確認

1. **etcd / Cilium 現状確認**
   - `node1-master` 上で `etcd Version: 3.5.26`（`3.6` 移行の必須前提 `>= 3.5.26` を満たす）、`cilium image (running): 1.18.6` を確認。
2. **etcd スナップショット取得（`node1-master`）**
   ```bash
   sudo ETCDCTL_API=3 /usr/local/bin/etcdctl \
     --endpoints=https://127.0.0.1:2379 \
     --cacert=/etc/ssl/etcd/ssl/ca.pem \
     --cert=/etc/ssl/etcd/ssl/member-node1-master.pem \
     --key=/etc/ssl/etcd/ssl/member-node1-master-key.pem \
     snapshot save /tmp/etcd-snapshot-pre-1.35.db
   ```
   - `HASH: e4873d6a` / `REVISION: 1087635765` / `TOTAL KEYS: 13316` / `109 MB` を取得しローカルへ `scp` 退避。
3. **Cilium Helm values 退避**
   - `helm get values cilium -n kube-system -a` をローカルに退避。

### 2.2 Kubespray `v2.31.0` チェックアウト & `venv` 再構築

```bash
cd kubespray/kubespray
git checkout v2.31.0
rm -rf venv
python3 -m venv venv
./venv/bin/pip install --upgrade pip
./venv/bin/pip install -r requirements.txt
```

- [.gitmodules](../.gitmodules) の `branch` を `release-2.30` → `release-2.31` に更新。
- Ansible がメジャー更新されたため `venv` を新規作成（`ansible==11.13.0` / `ansible-core 2.18.20` / `cryptography==46.0.7`）。

### 2.3 Inventory 更新

[inventory/](./inventory/) の `group_vars` を `v2.31.0` の `sample/group_vars` に同期（廃止された `ingress_nginx_*` / `dashboard_*` を [addons.yml](./inventory/group_vars/k8s_cluster/addons.yml) から除去）し、環境固有設定を適用：

- [k8s-cluster.yml](./inventory/group_vars/k8s_cluster/k8s-cluster.yml): `kube_version: 1.35.4`、`kube_owner: root`、`kube_network_plugin: cilium`
- [k8s-net-cilium.yml](./inventory/group_vars/k8s_cluster/k8s-net-cilium.yml):
  ```yaml
  cilium_extra_values:
    k8sServiceHost: "192.168.1.137"
    k8sServicePort: "6443"
    envoy:
      enabled: false
  ```
- [addons.yml](./inventory/group_vars/k8s_cluster/addons.yml): `metrics_server_enabled: true`

`kubespray/kubespray/inventory/mycluster` へ同期し、`ansible-inventory` および `--syntax-check` を通過させた。

---

## 3. Playbook 実行ログと発生事象

### 3.1 事前疎通確認（`playbooks/facts.yml`）

```bash
ansible-playbook -i inventory/mycluster/hosts.yaml \
  -u $USERNAME -b -v playbooks/facts.yml \
  --ask-become-pass | tee ../facts-1.35.log
```

```text
PLAY RECAP *********************************************************************
node1-master               : ok=33   changed=0    unreachable=0    failed=0    skipped=22   rescued=0    ignored=0
node2-worker               : ok=19   changed=0    unreachable=0    failed=0    skipped=17   rescued=0    ignored=0
nucboxg3-1                 : ok=19   changed=0    unreachable=0    failed=0    skipped=17   rescued=0    ignored=0
```

### 3.2 1 回目実行（`upgrade-cluster.yml`）と発生事象

Longhorn 以外の通常 Pod 退避のため `drain_timeout` はデフォルトの **6 分 (`360s`)** を維持しつつ、リトライ回数のみ `-e "drain_retries=0"` として実行：

```bash
ansible-playbook -i inventory/mycluster/hosts.yaml \
  -u $USERNAME -b -v upgrade-cluster.yml \
  -e kube_version=1.35.4 \
  -e "serial=1" \
  -e "drain_fallback_enabled=true" \
  -e "drain_retries=0" \
  --ask-become-pass | tee ../upgrade-1.35.log
```

```text
PLAY RECAP *********************************************************************
node1-master               : ok=524  changed=64   unreachable=0    failed=1    skipped=613  rescued=0    ignored=0
node2-worker               : ok=298  changed=23   unreachable=0    failed=0    skipped=235  rescued=0    ignored=0
nucboxg3-1                 : ok=294  changed=23   unreachable=0    failed=0    skipped=255  rescued=0    ignored=0
```

**発生した事象と対処：**
- `etcd`（`3.5.26` → `3.6.10`）および `node1-master` の `containerd`（`2.2.3`）更新は正常完了したが、`TASK [kubernetes/control-plane : Kubeadm | Upgrade first control plane node to 1.35.4]` の preflight チェック（`CreateJob`）が 15 秒タイムアウトして失敗。
- **原因**: `node1-master` にのみ過去に入った apt パッケージ `kubeadm`（`1.32.13-1.1`）由来の systemd drop-in **`/usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf`** が残っており、Kubespray 本来の `kubelet.service` を上書きして `/var/lib/kubelet/kubeadm-flags.env` の `--pod-infra-container-image=registry.k8s.io/pause:3.10` を渡していた。Kubernetes `v1.35` で同フラグが廃止されたため `kubelet v1.35.4` が起動失敗（`unknown flag: --pod-infra-container-image`）し、`kube-controller-manager` / `kube-scheduler` が停止していた。
- **対処**:
  ```bash
  # node1-master 上で競合 drop-in を退避して Kubespray 管理の kubelet.service で起動
  sudo mv /usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf /root/10-kubeadm.conf.bak
  sudo systemctl daemon-reload
  sudo systemctl restart kubelet

  # 失敗時に残った preflight Job を削除
  kubectl -n kube-system delete job upgrade-health-check-1795366034710
  ```
  これにより `kubelet`・`kube-apiserver`・`kube-controller-manager`・`kube-scheduler` がすべて `1/1 Running` に復帰した。

### 3.3 2 回目実行（リトライで完了）

```bash
ansible-playbook -i inventory/mycluster/hosts.yaml \
  -u $USERNAME -b -v upgrade-cluster.yml \
  -e kube_version=1.35.4 \
  -e "serial=1" \
  -e "drain_fallback_enabled=true" \
  -e "drain_retries=0" \
  --ask-become-pass | tee ../upgrade-1.35-2.log
```

```text
PLAY RECAP *********************************************************************
node1-master               : ok=589  changed=36   unreachable=0    failed=0    skipped=1043 rescued=0    ignored=0
node2-worker               : ok=444  changed=28   unreachable=0    failed=0    skipped=514  rescued=0    ignored=0
nucboxg3-1                 : ok=437  changed=28   unreachable=0    failed=0    skipped=537  rescued=0    ignored=0
```

- 所要時間 **40 分 54 秒**（`drain_retries=0` により前回の半分以下の時間で完了）で全ノード `failed=0`。
- 1 回目失敗時から `SchedulingDisabled` のままだった `node1-master` は 2 回目では既存 cordon 扱いとなり自動 uncordon されなかったため、完了後に手動で `kubectl uncordon node1-master` を実行して解除した。

---

## 4. 最終検証結果

### ノード・バージョン

```text
Server Version: v1.35.4
etcd Version: 3.6.10

NAME           STATUS   ROLES           AGE    VERSION   INTERNAL-IP     OS-IMAGE                         CONTAINER-RUNTIME
node1-master   Ready    control-plane   672d   v1.35.4   192.168.1.137   Debian GNU/Linux 12 (bookworm)   containerd://2.2.3
node2-worker   Ready    <none>          672d   v1.35.4   192.168.1.118   Debian GNU/Linux 12 (bookworm)   containerd://2.2.3
nucboxg3-1     Ready    <none>          462d   v1.35.4   192.168.1.99    Debian GNU/Linux 13 (trixie)     containerd://2.2.3
```

### Helm リリース

```text
NAME     	NAMESPACE      	REVISION	STATUS  	CHART          	APP VERSION
cilium   	kube-system    	7       	deployed	cilium-1.19.3  	1.19.3
librechat	librechat      	6       	deployed	librechat-3.2.1	v0.8.9
longhorn 	longhorn-system	6       	deployed	longhorn-1.12.2	v1.12.2
metallb  	metallb-system 	1       	deployed	metallb-0.14.9 	v0.14.9
traefik  	traefik        	2       	deployed	traefik-42.4.1 	v3.7.19
zot      	zot            	2       	deployed	zot-0.1.135    	v2.1.23
```

### コンポーネント・ワークロード確認

- **etcd**: `3.6.10` で正常稼働。
- **Cilium (`1.19.3`)**: 全 3 ノード `cilium` Pod `1/1 Running`（OOMKilled なし）、`cilium-operator` `2/2 Running`、`cilium-envoy` 未作成を確認。
- **kube-system**: Control Plane・CoreDNS (2/2)・nodelocaldns (3/3)・kube-proxy (3/3)・nginx-proxy (2/2)・metrics-server・external-dns すべて `Running`。
- **Longhorn (`v1.12.2`)**: 全 6 ボリューム `attached / healthy`。`node1-master` uncordon 後、全 3 ノードの `instance-manager` が `running` に復帰。
- **ワークロード**: `homelab-dns`, `librechat`, `metallb`, `traefik`, `wg-easy`, `zot` 正常稼働。

---

## 5. 今後の申し送り

- **ローカル `kubectl` 更新**: クライアントが `v1.32.0` のままサーバ `v1.35.4` と 3 マイナー差になっているため、`v1.35.x` への更新を推奨。
- **`node1-master` の不要 apt パッケージ**: 退避した `10-kubeadm.conf` の元になった apt パッケージ `kubeadm` (`1.32.13-1.1`) は Kubespray 管理環境では不要なため、将来的に `sudo apt remove kubeadm` しておくと apt 更新時に `10-kubeadm.conf` が復活するのを防げる。
- **Drain オプション**: `-e "serial=1" -e "drain_fallback_enabled=true" -e "drain_retries=0"`（`drain_timeout` デフォルト 6 分）で所要時間が大幅に短縮（約 41 分）され、非常に快適だった。
- **`kube-proxy` `nftables` モードへの移行（次回アップグレード時に実施）**: `v1.35` で `ipvs` モードが非推奨化（`v1.40` で削除予定）されたため、次回アップグレード時（`v1.36` 等）に [k8s-cluster.yml](./inventory/group_vars/k8s_cluster/k8s-cluster.yml) の `kube_proxy_mode: ipvs` を `kube_proxy_mode: nftables` へ変更する（全 3 ノードともカーネル 6.6 / 6.12 で要件 `>= 5.13` をクリア済み。切替後に古い `kube-ipvs0` をクリーンにするため、必要に応じノードを 1 台ずつ再起動する）。
