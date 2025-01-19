# kubespray inventory

Bootstrap raspberry pi cluster

## Environment

- Raspberry pi 5 Debian 12 (worker node)
- Raspberry pi 4 Debian 12 (control-plane node)

## Preparation

Add `cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory` to `/boot/firmware/cmdline.txt` to enable cgroup memory
```bash
root@node1-master:# cat /boot/firmware/cmdline.txt
console=serial0,115200 console=tty1 root=PARTUUID=8d931df5-02 rootfstype=ext4 fsck.repair=yes rootwait cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory
root@node1-master:# reboot now
```

## Construction
1. Copy inventory
   ```
   cp -rfp inventory kubespray/inventory/mycluster
   ```
1. Configure python venv
1. Run playbook (takes about 20mins)
   ```
   cd kubespray
   ansible-playbook -i inventory/mycluster/hosts.yaml -u $USERNAME -b -v cluster.yml --ask-become-pass --ask-pass
   ```

If something wrong happened, clean up the cluster by `reset.yml`

