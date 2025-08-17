## Adding Worker Node

See [official doc](https://github.com/kubernetes-sigs/kubespray/blob/master/docs/operations/nodes.md)

Note: debian 13 is not yet supported in official releases.
This procedure is tested with commit [155c1c1](https://github.com/kubernetes-sigs/kubespray/commit/155c1c153162baa9bc8c929b1e294d7b491ad4a8).

Replace `nucboxg3-1` with the name of newly added node.

```
cd kubesplay
source venv/bin/activate.fish
```

See https://github.com/kubernetes-sigs/kubespray/pull/12177

```
ansible-playbook -i inventory/mycluster/hosts.yaml -u $USERNAME -b -v playbooks/facts.yml --ask-become-pass --ask-pass
```

```
ansible-playbook -i inventory/mycluster/hosts.yaml -u $USERNAME -b -v scale.yml --ask-become-pass --ask-pass --limit=nucboxg3-1
```

```
PLAY RECAP *****************************************************************************************
nucboxg3-1                 : ok=435  changed=83   unreachable=0    failed=0    skipped=627  rescued=0    ignored=1

Sunday 17 August 2025  19:11:53 +0900 (0:00:00.018)       0:03:55.199 *********
===============================================================================
download : Download_container | Download image if required --------------------------------- 17.38s
download : Download_container | Download image if required --------------------------------- 16.93s
download : Download_file | Download item ---------------------------------------------------- 8.33s
download : Download_file | Download item ---------------------------------------------------- 7.72s
download : Download_container | Download image if required ---------------------------------- 5.96s
download : Download_container | Download image if required ---------------------------------- 5.48s
container-engine/containerd : Download_file | Download item --------------------------------- 5.12s
download : Download_file | Download item ---------------------------------------------------- 4.45s
container-engine/validate-container-engine : Populate service facts ------------------------- 4.15s
download : Download_container | Download image if required ---------------------------------- 3.57s
container-engine/runc : Download_file | Download item --------------------------------------- 3.54s
container-engine/nerdctl : Download_file | Download item ------------------------------------ 3.43s
container-engine/crictl : Download_file | Download item ------------------------------------- 3.33s
kubernetes/kubeadm : Create kubeadm token for joining nodes with 24h expiration (default) --- 3.13s
download : Extract_file | Unpacking archive ------------------------------------------------- 2.80s
container-engine/crictl : Extract_file | Unpacking archive ---------------------------------- 2.69s
container-engine/containerd : Containerd | Unpack containerd archive ------------------------ 2.63s
network_plugin/cni : CNI | Copy cni plugins ------------------------------------------------- 2.62s
etcd : Gen_certs | run cert generation script for all clients ------------------------------- 2.60s
container-engine/nerdctl : Extract_file | Unpacking archive --------------------------------- 2.52s
```

```bash
$ k get node
NAME           STATUS   ROLES           AGE     VERSION
node1-master   Ready    control-plane   209d    v1.31.4
node2-worker   Ready    <none>          209d    v1.31.4
nucboxg3-1     Ready    <none>          8m53s   v1.31.4
```

### Issue: cilium pod crashes

```
time="2025-08-17T10:22:53Z" level=info msg="Waiting until local node addressing before starting watchers depending on it" subsys=k8s-watcher
time="2025-08-17T10:22:53Z" level=error msg="Unable to read etcd configuration." error="open /etc/cilium/certs/cert.crt: no such file or directory" subsys=kvstore
time="2025-08-17T10:22:53Z" level=info msg="Creating etcd client" ConfigPath=/var/lib/etcd-config/etcd.config KeepAliveHeartbeat=15s KeepAliveTimeout=25s ListLimit=256 MaxInflight=20 RateLimit=20 subsys=kvstore
time="2025-08-17T10:22:53Z" level=info msg="Waiting for all etcd configuration files to be available" error="open /etc/cilium/certs/cert.crt: no such file or directory" subsys=kvstore
time="2025-08-17T10:22:53Z" level=info msg="Node updated" clusterName=default nodeName=node1-master subsys=nodemanager
time="2025-08-17T10:22:53Z" level=info msg="Node updated" clusterName=default nodeName=node2-worker subsys=nodemanager
time="2025-08-17T10:22:58Z" level=info msg="Waiting for all etcd configuration files to be available" error="open /etc/cilium/certs/cert.crt: no such file or directory" subsys=kvstore
time="2025-08-17T10:23:03Z" level=info msg="Waiting for all etcd configuration files to be available" error="open /etc/cilium/certs/cert.crt: no such file or directory" subsys=kvstore
time="2025-08-17T10:23:08Z" level=info msg="Waiting for all etcd configuration files to be available" error="open /etc/cilium/certs/cert.crt: no such file or directory" subsys=kvstore
time="2025-08-17T10:23:13Z" level=info msg="Waiting for all etcd configuration files to be available" error="open /etc/cilium/certs/cert.crt: no such file or directory" subsys=kvstore
```

Seems fails to load /etc/cilium/certs (hostPath) and I found that the directory is empty on the new node.
[This task](https://github.com/kubernetes-sigs/kubespray/blob/51764b208b328b539193e766dd062050c998d714/roles/network_plugin/cilium/tasks/install.yml#L11) seems to copy cert files to this directory but idk if this is already run for the new node.

As a workaround, I manually copied the content of /etc/cilium/certs from node2-worker.

```bash
$ k get po -n kube-system -o wide | grep nucbox
cilium-wbqb9                           1/1     Running   0                52s    192.168.1.99    nucboxg3-1     <none>           <none>
kube-proxy-cnf5b                       1/1     Running   0                70m    192.168.1.99    nucboxg3-1     <none>           <none>
nginx-proxy-nucboxg3-1                 1/1     Running   0                70m    192.168.1.99    nucboxg3-1     <none>           <none>
nodelocaldns-678hx                     1/1     Running   0                70m    192.168.1.99    nucboxg3-1     <none>           <none>
```

It's now working fine.
