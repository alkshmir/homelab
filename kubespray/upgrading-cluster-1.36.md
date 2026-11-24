# Upgrading Cluster to Kubernetes v1.36.4 (Kubespray v2.32.0)

2026-11-24 実施。Kubespray を `v2.32.0` へ更新し、Kubernetes を `v1.35.4` から `v1.36.4` へアップグレード、および前回申し送りの `kube-proxy` `nftables` モード移行を実施した作業ログ。

- 前回のアップグレード記録: [upgrading-cluster-1.35.md](./upgrading-cluster-1.35.md)

---

## 1. 対象構成とバージョン変更結果

| ノード名 | IP | ハードウェア / OS | 役割 |
| :--- | :--- | :--- | :--- |
| `node1-master` | `192.168.1.137` | Raspberry Pi 4 / Debian 12 (arm64) | `kube_control_plane`, `etcd`, `kube_node` |
| `node2-worker` | `192.168.1.118` | Raspberry Pi 5 / Debian 12 (arm64) | `kube_node` |
| `nucboxg3-1` | `192.168.1.99` | NucBox G3 / Debian 13 Trixie (amd64) | `kube_node` |

| コンポーネント | アップグレード前 | アップグレード後 | 備考 |
| :--- | :--- | :--- | :--- |
| Kubespray | `v2.31.0` (`1c9add489`) | **`v2.32.0`** (`9751ea6f6`) | `ansible==12.3.0` (`ansible-core 2.19.15`) |
| Kubernetes | `v1.35.4` | **`v1.36.4`** | 全 3 ノード更新完了（ローカル `kubectl v1.37.2` とのスキュー警告解消） |
| etcd | `3.6.10` | **`3.6.14`** | `etcd_deployment_type: host` |
| Cilium | `1.19.3` | **`1.20.2`** | `cilium_extra_values` 維持、`cilium-envoy` 抑止済み |
| containerd | `2.2.3` | **`2.3.5`** | 全 3 ノード更新完了（`config.toml` `version = 4` 自動移行） |
| runc | `1.4.2` | **`1.4.3`** | 全 3 ノード更新完了 |
| CoreDNS | `v1.12.4` | **`v1.14.2`** | |
| DNS autoscaler | `v1.8.8` | **`1.10.3`** | |
| metrics-server | `v0.8.1` | **`0.9.0`** | |
| NodeLocal DNSCache | `1.25.0` | `1.25.0` | 変更なし |
| `kube-proxy` モード | `ipvs` | **`nftables`** | アップグレード完了後に切替実施、`kube-ipvs0` 自動削除済み |
| Longhorn | `v1.12.2` | **`v1.12.2`** | K8s `v1.36.4+` 向け修正 (`#14218`) 込みのため維持 |

---

## 2. 事前作業・準備

### 2.1 事前障害復旧（Phase 0）

1. **`node1-master` の `/opt/cni/bin/` 復旧**
   - 2026-11-23 の `apt autoremove` により過去の apt パッケージ `kubernetes-cni (1.6.0-1.1)` が削除され、`/opt/cni/bin/` から `loopback` 等の標準 CNI バイナリが欠損（`cilium-cni` のみ残存）していた。この影響で `coredns` や `longhorn-driver-deployer` 等 13 Pod が `ContainerCreating` / `CrashLoopBackOff` でスタックしていた。
   - `node1-master` 上の既存キャッシュ `/tmp/releases/cni-plugins-linux-arm64-1.9.1.tgz` から `/opt/cni/bin/` へ展開・所有権を `root:root` に統一したところ、スタックしていた全 13 Pod が即時 `Running` へ復帰した。
2. **`node2-worker` の電源復帰**
   - 2026-11-24 00:26 JST ごろから `NotReady`（ping / SSH 不通）となっていた `node2-worker`（Raspberry Pi 5）の電源を再投入し、全 3 ノード `Ready` および Longhorn 全 6 ボリューム `attached / healthy` の回復を確認した。

### 2.2 バックアップ取得（Phase 1）

1. **etcd スナップショット取得（`node1-master`）**
   ```bash
   sudo ETCDCTL_API=3 /usr/local/bin/etcdctl \
     --endpoints=https://127.0.0.1:2379 \
     --cacert=/etc/ssl/etcd/ssl/ca.pem \
     --cert=/etc/ssl/etcd/ssl/member-node1-master.pem \
     --key=/etc/ssl/etcd/ssl/member-node1-master-key.pem \
     snapshot save /tmp/etcd-snapshot-pre-1.36.db

   sudo /usr/local/bin/etcdutl snapshot status /tmp/etcd-snapshot-pre-1.36.db -w table
   ```
   - `HASH: 1265820a` / `REVISION: 1090001734` / `TOTAL KEYS: 1354` / `116 MB`（`VERSION: 3.6.0`）を取得しローカル `/tmp/etcd-snapshot-pre-1.36.db` へ `scp` 退避（etcd 3.6 系では `snapshot status` は `etcdutl` を使用）。
2. **Cilium Helm values & Longhorn 状態退避**
   - `helm get values cilium -n kube-system -a` → `/tmp/cilium-values-pre-1.36.yaml`
   - `kubectl -n longhorn-system get volumes.longhorn.io,settings.longhorn.io -o yaml` → `/tmp/longhorn-pre-1.36.yaml`

### 2.3 Kubespray `v2.32.0` チェックアウト & `venv` 再構築（Phase 2）

- [.gitmodules](../.gitmodules) の `branch` を `release-2.31` → `release-2.32` に更新。
- 制御ホスト（macOS）の `pyenv` グローバルが `3.15.2`（`ansible-core 2.19` の制御ノード対応は Python `3.11`〜`3.13`）のため、`Python 3.12.1` を明示指定して `venv` を再構築した：
  ```bash
  cd kubespray/kubespray
  git fetch --tags
  git checkout v2.32.0
  rm -rf venv
  ~/.pyenv/versions/3.12.1/bin/python3 -m venv venv
  ./venv/bin/pip install --upgrade pip
  ./venv/bin/pip install -r requirements.txt
  ```
- 導入確認: `Python 3.12.1` / `ansible==12.3.0` / `ansible-core==2.19.15` / `cryptography==50.0.1`。

### 2.4 Inventory 更新（Phase 3）

[inventory/](./inventory/) を `v2.32.0` の `inventory/sample` に同期し、環境固有設定を適用：

- `group_vars/k8s_cluster/kube_control_plane.yml` を [group_vars/kube_control_plane.yml](./inventory/group_vars/kube_control_plane.yml) へ移動（PR `#13518`）。
- [k8s-cluster.yml](./inventory/group_vars/k8s_cluster/k8s-cluster.yml): `kube_version: 1.36.4`、削除された `# eviction_hard_control_plane: {}` 行を除去（PR `#13400`）。`kube_owner: root`、`kube_network_plugin: cilium` を維持（`kube_proxy_mode` は一旦 `ipvs` のままアップグレードし、完了後に `nftables` へ切替）。
- [k8s-net-cilium.yml](./inventory/group_vars/k8s_cluster/k8s-net-cilium.yml): `cilium_extra_values`（`k8sServiceHost: "192.168.1.137"`, `k8sServicePort: "6443"`, `envoy.enabled: false`）を維持。
- [addons.yml](./inventory/group_vars/k8s_cluster/addons.yml)（`metrics_server_enabled: true` 維持）・`offline.yml`・`openstack.yml`・`k8s-net-custom-cni.yml` を同期。
- `rsync -av --delete --exclude='credentials/'` で `kubespray/kubespray/inventory/mycluster/` へ同期し、`ansible-inventory` および `--syntax-check` を通過させた。

---

## 3. Playbook 実行ログと発生事象

### 3.1 事前疎通確認（`playbooks/facts.yml`）

```bash
./venv/bin/ansible-playbook -i inventory/mycluster/hosts.yaml \
  -u $USERNAME -b -v playbooks/facts.yml \
  --ask-become-pass | tee ../facts-1.36.log
```

```text
PLAY RECAP *********************************************************************
node1-master               : ok=34   changed=0    unreachable=0    failed=0    skipped=21   rescued=0    ignored=0
node2-worker               : ok=21   changed=0    unreachable=0    failed=0    skipped=16   rescued=0    ignored=0
nucboxg3-1                 : ok=21   changed=0    unreachable=0    failed=0    skipped=16   rescued=0    ignored=0
```

### 3.2 1 回目実行（`upgrade-1.36.log`）と発生事象

`ansible-core 2.19` の厳密な bool 評価に対応するため、`-e` オプションは JSON 形式で指定して実行：

```bash
./venv/bin/ansible-playbook -i inventory/mycluster/hosts.yaml \
  -u $USERNAME -b -v upgrade-cluster.yml \
  -e '{"kube_version": "1.36.4", "serial": 1, "drain_fallback_enabled": true, "drain_retries": 0}' \
  --ask-become-pass | tee ../upgrade-1.36.log
```

```text
PLAY RECAP *********************************************************************
node1-master               : ok=179  changed=13   unreachable=0    failed=1    skipped=245  rescued=0    ignored=0
node2-worker               : ok=150  changed=14   unreachable=0    failed=0    skipped=105  rescued=0    ignored=0
nucboxg3-1                 : ok=109  changed=4    unreachable=0    failed=0    skipped=76   rescued=0    ignored=0
```

- **事象**: `TASK [download : Download_container | Download image if required]` で `node1-master` での `nerdctl -n k8s.io pull quay.io/cilium/cilium:v1.20.2` が失敗（所要 11 分 18 秒）。
  ```text
  failed to get stream processor for application/vnd.oci.image.layer.v1.tar+gzip: fork/exec /usr/bin/unpigz: no such file or directory
  ```
- **原因**: 前日の `apt autoremove` により `kubernetes-cni` とあわせて `pigz` も削除されており、起動中の `containerd` が `/usr/bin/unpigz` を呼び出せず失敗していた。
- **対処**: `node1-master` へ `sudo apt-get install -y pigz` で `pigz 2.6` を再導入し、`nerdctl -n k8s.io pull quay.io/cilium/cilium:v1.20.2` の成功を確認した。

### 3.3 2 回目実行（`upgrade-1.36-2.log`）と発生事象

```bash
./venv/bin/ansible-playbook -i inventory/mycluster/hosts.yaml \
  -u $USERNAME -b -v upgrade-cluster.yml \
  -e '{"kube_version": "1.36.4", "serial": 1, "drain_fallback_enabled": true, "drain_retries": 0}' \
  --ask-become-pass | tee ../upgrade-1.36-2.log
```

```text
PLAY RECAP *********************************************************************
node1-master               : ok=612  changed=72   unreachable=0    failed=0    skipped=855  rescued=0    ignored=0
node2-worker               : ok=459  changed=38   unreachable=0    failed=0    skipped=460  rescued=0    ignored=0
nucboxg3-1                 : ok=343  changed=26   unreachable=0    failed=1    skipped=341  rescued=1    ignored=0
```

- **事象**: `node1-master` / `node2-worker` の `v1.36.4`・`containerd 2.3.5`・`etcd 3.6.14`・`Cilium 1.20.2`・`CoreDNS v1.14.2`・`metrics-server 0.9.0`・`dns-autoscaler 1.10.3` 更新はすべて正常完了したが、最終ノード `nucboxg3-1` の `TASK [upgrade/pre-upgrade : Drain node]` が以下で失敗（所要 56 分 35 秒、rescue により `nucboxg3-1` は自動 uncordon 済み）：
  ```text
  fatal: [nucboxg3-1 -> node1-master(192.168.1.137)]: FAILED! =>
    "module_stderr": "/bin/sh: 1: /usr/bin/python3.13: not found"
  ```
- **原因**: Ansible 2.19 で `async` + `delegate_to` タスクを実行した際、委譲元 `nucboxg3-1`（Debian 13 / 自動検出 `/usr/bin/python3.13`）のインタープリタパスが委譲先 `node1-master`（Debian 12 / Python `3.11` のみ）に適用されていた。
- **対処**: [hosts.yaml](./inventory/hosts.yaml) の `all.vars` に全 3 ノード共通で存在する `ansible_python_interpreter: /usr/bin/python3` を明示追加し、`mycluster/` へ同期した。

### 3.4 3 回目実行（`upgrade-1.36-3.log`・`nucboxg3-1` 完了）

`node1-master` / `node2-worker` と主要アドオンは更新済みのため、`facts.yml` で全ノードのファクトを再取得したうえで `--limit nucboxg3-1`（＋ `"drain_timeout": "60s"`）で残りの `nucboxg3-1` のみ実行：

```bash
./venv/bin/ansible-playbook -i inventory/mycluster/hosts.yaml \
  -u $USERNAME -b -v playbooks/facts.yml \
  --ask-become-pass && \
./venv/bin/ansible-playbook -i inventory/mycluster/hosts.yaml \
  -u $USERNAME -b -v upgrade-cluster.yml \
  --limit nucboxg3-1 \
  -e '{"kube_version": "1.36.4", "serial": 1, "drain_fallback_enabled": true, "drain_retries": 0, "drain_timeout": "60s"}' \
  --ask-become-pass | tee ../upgrade-1.36-3.log
```

```text
PLAY RECAP *********************************************************************
nucboxg3-1                 : ok=431  changed=27   unreachable=0    failed=0    skipped=540  rescued=0    ignored=0
```

- 所要時間 **4 分 23 秒**で `failed=0` 完了。`Drain node` も 54 秒で正常通過し、全 3 ノードが `v1.36.4` / `containerd://2.3.5` となった。

---

## 4. `kube-proxy` `nftables` モード移行（Phase 7）

`v1.36.4` の正常稼働を確認後、前回申し送りの `kube-proxy` `ipvs` → `nftables` 移行を実施した。

1. **バックアップ & Inventory 更新**
   - `cm/kube-proxy`・`cm/kubeadm-config`・`node1-master:/etc/kubernetes/kubeadm-config.yaml` を `/tmp/` へ退避。
   - [k8s-cluster.yml](./inventory/group_vars/k8s_cluster/k8s-cluster.yml) の `kube_proxy_mode` を `ipvs` → **`nftables`** に変更し、`mycluster/` へ同期。
2. **クラスタ反映 & `kube-proxy` ローリング再起動**
   - `node1-master` の `/etc/kubernetes/kubeadm-config.yaml` を `mode: nftables` に更新し、`sudo /usr/local/bin/kubeadm init phase addon kube-proxy --config /etc/kubernetes/kubeadm-config.yaml` を実行。
   - `node1-master` / `node2-worker` に `/etc/modules-load.d/nf_tables.conf` を配置（`nucboxg3-1` は Debian 13 で `nf_tables` ロード済み）。
   - `kubectl -n kube-system rollout restart ds/kube-proxy` を実施し、3/3 正常ロールアウト。
3. **移行確認**
   - 全 3 ノードの `kube-proxy` ログで **`Using nftables Proxier`**、コンテナ内 `nft list tables` で **`table ip kube-proxy` / `table ip6 kube-proxy`** の生成を確認。
   - 全 3 ノードで **`kube-ipvs0` が自動削除済み**（`Device "kube-ipvs0" does not exist.`）および `ipvsadm -Ln` 仮想サーバ 0 件を確認（緊急のノード再起動は不要）。
   - 疎通テスト: ClusterIP（`librechat` Pod → `https://10.233.0.1:443/version` → `200`）、クラスタ内 DNS（`kubernetes.default.svc.cluster.local` → `10.233.0.1`）、MetalLB LoadBalancer（Longhorn `200` / wg-easy `302` / Traefik `404` / homelab-dns 名前解決正常）すべて OK。

---

## 5. 最終検証結果

### ノード・バージョン

```text
Client Version: v1.37.2
Server Version: v1.36.4
etcd Version: 3.6.14

NAME           STATUS   ROLES           AGE    VERSION   INTERNAL-IP     OS-IMAGE                         CONTAINER-RUNTIME
node1-master   Ready    control-plane   673d   v1.36.4   192.168.1.137   Debian GNU/Linux 12 (bookworm)   containerd://2.3.5
node2-worker   Ready    <none>          673d   v1.36.4   192.168.1.118   Debian GNU/Linux 12 (bookworm)   containerd://2.3.5
nucboxg3-1     Ready    <none>          463d   v1.36.4   192.168.1.99    Debian GNU/Linux 13 (trixie)     containerd://2.3.5
```

### Helm リリース

```text
NAME     	NAMESPACE      	REVISION	STATUS  	CHART          	APP VERSION
cilium   	kube-system    	8       	deployed	cilium-1.20.2  	1.20.2
librechat	librechat      	6       	deployed	librechat-3.2.1	v0.8.9
longhorn 	longhorn-system	6       	deployed	longhorn-1.12.2	v1.12.2
metallb  	metallb-system 	1       	deployed	metallb-0.14.9 	v0.14.9
traefik  	traefik        	2       	deployed	traefik-42.4.1 	v3.7.19
zot      	zot            	2       	deployed	zot-0.1.135    	v2.1.23
```

### コンポーネント・ワークロード確認

- **etcd**: `3.6.14` 正常稼働。
- **containerd / runc**: 全 3 ノード `containerd 2.3.5`（`config.toml` `version = 4`）・`runc 1.4.3`・`/opt/cni/bin`（`root:root`, 21 ファイル）。
- **Cilium (`1.20.2`)**: `cilium` 3/3、`cilium-operator` 2/2、`cilium-envoy` 未作成。
- **kube-system**: Control Plane・CoreDNS `v1.14.2` (2/2)・nodelocaldns (3/3)・kube-proxy `v1.36.4` (`nftables`, 3/3)・metrics-server `0.9.0`・dns-autoscaler `1.10.3`・external-dns すべて `Running`。
- **Longhorn (`v1.12.2`)**: 全 6 ボリューム `attached / healthy`、`instance-manager` 3/3、`longhorn-driver-deployer` 1/1。
- **ワークロード**: `gpt-bot`, `homelab-dns`, `librechat`, `metallb`, `traefik`, `wg-easy` (`nucboxg3-1` 固定), `zot` すべて `Running`（全 Namespace で異常 Pod 0 件）。

---

## 7. 今後の申し送り

- **`ansible_python_interpreter: /usr/bin/python3`**: Debian 12 (`python3.11`) と Debian 13 (`python3.13`) の混在環境で Ansible 2.19 の `async` + `delegate_to` が失敗するため、[hosts.yaml](./inventory/hosts.yaml) の `all.vars` 設定を維持すること。
- **Ansible 2.19 以降の `-e` 指定**: bool / 数値の extra-vars は必ず JSON 形式（`-e '{"drain_fallback_enabled": true, ...}'`）で渡すこと。また制御ホストの `venv` 作成時は `ansible-core` 対応の Python バージョン（今回は `3.12.1`）を明示すること。
- **`node2-worker` の残存 apt パッケージ**: `kubectl (1.32.13-1.1)` / `kubernetes-cni (1.6.0-1.1)` が残っているため、将来 `apt remove` / `autoremove` を実行する際は `/opt/cni/bin/` および `pigz` が消えないよう注意すること（消えた場合は直後に CNI プラグイン再展開・`pigz` 維持を実施）。
- **Debian 12 ホストでの `nft` コマンド**: Debian 12 の `/usr/sbin/nft` (`v1.0.6`) は `kube-proxy` コンテナ内の新しい `nft` が生成したルールを参照すると segfault するため、ルール確認は `kubectl -n kube-system exec ds/kube-proxy -- nft list table ip kube-proxy` を使用すること。
- **今後の保守検討項目**: MetalLB（現行 `v0.14.9` → `v0.16.x`）、Longhorn `v1.13.x`（`#14218` 修正取り込み版リリース後）、ノード `/etc/resolv.conf` の nameserver 数整理（`DNSConfigForming` 警告解消）、不要になった退避ファイル（`/tmp/etcd-snapshot-pre-1.36.db` 等）の削除。
