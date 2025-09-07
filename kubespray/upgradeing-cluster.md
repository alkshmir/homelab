## Upgrading Cluster

Upgrade kubernetes from v1.31.4 to ~~v1.32.8~~ v1.32.6

```
$ k version
Client Version: v1.32.8
Kustomize Version: v5.5.0
Server Version: v1.31.4
```

update `kube_version` var in `inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml`

```
cd kubesplay
source venv/bin/activate.fish
```

<details>
<summary>failed command</summary>
```
ansible-playbook -i inventory/mycluster/hosts.yaml -u $USERNAME -b -v upgrade-cluster.yml -e kube_version=1.32.8 -e "serial=1" --ask-become-pass --ask-pass
```

Failed as roles/kubesplay_defaults/vars/main/checksums.yml does not contain 1.32.8 yet in [155c1c1](https://github.com/kubernetes-sigs/kubespray/commit/155c1c153162baa9bc8c929b1e294d7b491ad4a8).

I use 1.32.6 instead.
</details>

```
ansible-playbook -i inventory/mycluster/hosts.yaml -u $USERNAME -b -v upgrade-cluster.yml -e kube_version=1.32.6 -e "serial=1" --ask-become-pass --ask-pass
```

```
TASK [network_plugin/cilium : Cilium | Install] *****************************************************************************************************
fatal: [node1-master]: FAILED! => {"changed": true, "cmd": ["/usr/local/bin/cilium", "install", "--version", "1.17.3", "-f", "/etc/kubernetes/cilium-values.yaml", "-f", "/etc/kubernetes/cilium-extra-values.yaml"], "delta": "0:00:27.264050", "end": "2025-08-31 07:43:35.938426", "msg": "non-zero return code", "rc": 1, "start": "2025-08-31 07:43:08.674376", "stderr": "\nError: Unable to install Cilium: Unable to continue with install: ServiceAccount \"cilium\" in namespace \"kube-system\" exists and cannot be imported into the current release: invalid ownership metadata; label validation error: missing key \"app.kubernetes.io/managed-by\": must be set to \"Helm\"; annotation validation error: missing key \"meta.helm.sh/release-name\": must be set to \"cilium\"; annotation validation error: missing key \"meta.helm.sh/release-namespace\": must be set to \"kube-system\"", "stderr_lines": ["", "Error: Unable to install Cilium: Unable to continue with install: ServiceAccount \"cilium\" in namespace \"kube-system\" exists and cannot be imported into the current release: invalid ownership metadata; label validation error: missing key \"app.kubernetes.io/managed-by\": must be set to \"Helm\"; annotation validation error: missing key \"meta.helm.sh/release-name\": must be set to \"cilium\"; annotation validation error: missing key \"meta.helm.sh/release-namespace\": must be set to \"kube-system\""], "stdout": "ℹ️  Using Cilium version 1.17.3\nℹ️  Using cluster name \"default\"\n🔮 Auto-detected kube-proxy has been installed", "stdout_lines": ["ℹ️  Using Cilium version 1.17.3", "ℹ️  Using cluster name \"default\"", "🔮 Auto-detected kube-proxy has been installed"]}

```

なにこれ

よくわからんけどciliumのvarを上書き

```
cp sample/group_vars/k8s_cluster/k8s-net-cilium.yml mycluster/group_vars/k8s_cluster/k8s-net-cilium.yml 
```
https://github.com/kubernetes-sigs/kubespray/issues/12252

これは`-e "cilium_remove_old_resources=true"`で治った

なんかlonghornのpodがevictできなくてnodeがdrainできないみたいなので、`--disable-eviction`でリトライしてみる

```
ansible-playbook -i inventory/mycluster/hosts.yaml -u $USERNAME -b -v upgrade-cluster.yml -e kube_version=1.32.6 -e "serial=1" -e "cilium_remove_old_resources=true" -e "drain_fallback_enabled=true" --ask-become-pass --ask-pass
```

Ciliumが全部落ちて先に進まなくなった。
もう何もわからない。
```
$ k get po -n kube-system     -o wide
NAME                                   READY   STATUS                  RESTARTS         AGE     IP              NODE           NOMINATED NODE   READINESS GATES
cilium-envoy-28phn                     1/1     Running                 0                174m    192.168.1.118   node2-worker   <none>           <none>
cilium-envoy-95l79                     0/1     CrashLoopBackOff        38 (4m6s ago)    174m    192.168.1.137   node1-master   <none>           <none>
cilium-envoy-ldw4v                     1/1     Running                 0                174m    192.168.1.99    nucboxg3-1     <none>           <none>
cilium-gh8bl                           0/1     Init:CrashLoopBackOff   38 (3m35s ago)   174m    192.168.1.137   node1-master   <none>           <none>
cilium-operator-5bdf79db87-b552b       1/1     Running                 0                111m    192.168.1.118   node2-worker   <none>           <none>
cilium-operator-5bdf79db87-fzvnw       1/1     Running                 0                24m     192.168.1.99    nucboxg3-1     <none>           <none>
cilium-q4jdf                           0/1     Init:CrashLoopBackOff   43 (111s ago)    174m    192.168.1.99    nucboxg3-1     <none>           <none>
cilium-qwg4d                           0/1     Init:CrashLoopBackOff   43 (2m45s ago)   174m    192.168.1.118   node2-worker   <none>           <none>
coredns-6fdd44cdfd-89whr               0/1     Pending                 0                99m     <none>          <none>         <none>           <none>
coredns-6fdd44cdfd-hpxjb               0/1     Pending                 0                99m     <none>          <none>         <none>           <none>
dns-autoscaler-56cb45595c-c2js9        0/1     Pending                 0                167m    <none>          <none>         <none>           <none>
external-dns-5f87769b4c-4bqjb          0/1     Pending                 0                167m    <none>          <none>         <none>           <none>
kube-apiserver-node1-master            1/1     Running                 0                3h52m   192.168.1.137   node1-master   <none>           <none>
kube-controller-manager-node1-master   1/1     Running                 5 (24m ago)      3h52m   192.168.1.137   node1-master   <none>           <none>
kube-proxy-6l4c5                       1/1     Running                 0                100m    192.168.1.137   node1-master   <none>           <none>
kube-proxy-gkxsj                       1/1     Running                 0                100m    192.168.1.99    nucboxg3-1     <none>           <none>
kube-proxy-tn28n                       1/1     Running                 0                100m    192.168.1.118   node2-worker   <none>           <none>
kube-scheduler-node1-master            1/1     Running                 4 (24m ago)      3h52m   192.168.1.137   node1-master   <none>           <none>
metrics-server-5dff58bc89-c5zg5        0/1     Pending                 0                99m     <none>          <none>         <none>           <none>
nginx-proxy-node2-worker               1/1     Running                 0                100m    192.168.1.118   node2-worker   <none>           <none>
nginx-proxy-nucboxg3-1                 1/1     Running                 0                73m     192.168.1.99    nucboxg3-1     <none>           <none>
nodelocaldns-dfxdz                     1/1     Running                 0                3h50m   192.168.1.99    nucboxg3-1     <none>           <none>
nodelocaldns-ljs4h                     1/1     Running                 0                3h51m   192.168.1.137   node1-master   <none>           <none>
nodelocaldns-xf8fs                     1/1     Running                 2 (3h50m ago)    3h50m   192.168.1.118   node2-worker   <none>           <none>

```

https://github.com/kubernetes-sigs/kubespray/issues/12276

kube_ownerをrootに変えて再実行するとうまくいったが、cilium-envoyがnode1-masterだけcrashLoopbackOffする

### cilium-envoy

```
external/com_github_google_tcmalloc/tcmalloc/system-alloc.cc:625] MmapAligned() failed - unable to allocate with tag (hint, size, alignment) - is something limiting address placement? 0x17e1c0000000 1073741824 1073741824 @ 0x5567c24ec4 0x5567c21300 0x5567c20c08 0x5567c00210 0x5567c1e900 0x5567c1e6d4 0x5567bf59c8 0x5567b083d4 0x5567b050f0 0x7fae498614
external/com_github_google_tcmalloc/tcmalloc/arena.cc:58] FATAL ERROR: Out of memory trying to allocate internal tcmalloc data (bytes, object-size); is something preventing mmap from succeeding (sandbox, VSS limitations)? 131072 632 @ 0x5567c25234 0x5567c002a0 0x5567c1e900 0x5567c1e6d4 0x5567bf59c8 0x5567b083d4 0x5567b050f0 0x7fae498614```
```

https://github.com/envoyproxy/envoy/issues/23339

このIssueによると、envoyはカーネルの設定が対応してないからraspberry pi OS on raspberry pi 4で起動できない（？）と書かれている気がした。

そもそも[cilium-envoy](https://docs.cilium.io/en/latest/security/network/proxy/envoy/)って何？って調べたが、これをdisableするとDaemonSetではなくcilium Podの中で別プロセスとしてEnvoyを起動するらしい。

> When Cilium L7 functionality (Ingress, Gateway API, Network Policies with L7 functionality, L7 Protocol Visibility) is enabled or installed in a Kubernetes cluster, the Cilium agent starts an Envoy proxy as separate process within the Cilium agent pod.
> 
> That Envoy proxy instance becomes responsible for proxying all matching L7 requests on that node. As a result, L7 traffic targeted by policies depends on the availability of the Cilium agent pod.
> 
> Alternatively, it’s possible to deploy the Envoy proxy as independently life-cycled DaemonSet called `cilium-envoy` instead of running it from within the Cilium Agent Pod.

前はこんなDaemonSetなかった気がするのと、OS再インストールはあまりやりたくないのでcilium Podの中に入れることにした。

> To enable the dedicated Envoy proxy DaemonSet, install Cilium with the Helm value `envoy.enabled` set to `true.`

[kubesprayのtemplate](https://github.com/kubernetes-sigs/kubespray/blob/v2.28.1/roles/network_plugin/cilium/templates/values.yaml.j2)
を見るとenvoy.enabledがconfigurableでなさそう。
cilium以外動いているので、他のansibleを動かすのは避けたい(30分かかるし)

kubesprayでもhelmでインストールされてそうだったので、valuesを更新してciliumのchartをupgradeすることにした

まずdry run
``` 
helm upgrade cilium cilium/cilium                                                                                
    --namespace kube-system \
    --reuse-values \
    --set envoy.enabled=false \
    --version 1.17.3 \
    --dry-run \
    --debug
```

`envoy.enabled`が`false`になってそうだったので、

``` 
helm upgrade cilium cilium/cilium                                                                                
    --namespace kube-system \
    --reuse-values \
    --set envoy.enabled=false \
    --version 1.17.3
```

これでcilium-envoyが消えてようやく全て動いた。

kubesprayでciliumインストールは毎回トラブルになるので、やめようかな…
