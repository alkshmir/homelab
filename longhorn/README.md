# Longhorn

## Preparation
- install open-iscsi for each node
  ```bash
  sudo apt-get install open-iscsi
  sudo modprobe iscsi_tcp
  ```

- install cryptsetup for each node
  ```bash
  sudo apt-get install cryptsetup
  ```

- [MetalLB is set up](../metallb)

## Deployment

```bash
helm repo add longhorn https://charts.longhorn.io
helm repo update
helm install longhorn longhorn/longhorn --namespace longhorn-system --create-namespace --version 1.7.2 --values values.yaml
```

```bash
$ k get po -n longhorn-system
NAME                                                READY   STATUS    RESTARTS        AGE
csi-attacher-698944d5b-4m8rj                        1/1     Running   1 (7m35s ago)   10m
csi-attacher-698944d5b-lt56m                        1/1     Running   3 (7m21s ago)   10m
csi-attacher-698944d5b-twsl6                        1/1     Running   1 (9m21s ago)   10m
csi-provisioner-b98c99578-9w9hm                     1/1     Running   1 (7m13s ago)   10m
csi-provisioner-b98c99578-h5cvw                     1/1     Running   0               10m
csi-provisioner-b98c99578-t5kvk                     1/1     Running   1 (8m40s ago)   10m
csi-resizer-7474b7b598-44489                        1/1     Running   0               10m
csi-resizer-7474b7b598-4k4j5                        1/1     Running   0               10m
csi-resizer-7474b7b598-6nfz4                        1/1     Running   0               10m
csi-snapshotter-774467fdc7-k9zzk                    1/1     Running   1 (8m40s ago)   10m
csi-snapshotter-774467fdc7-mfr2w                    1/1     Running   0               10m
csi-snapshotter-774467fdc7-xzfdn                    1/1     Running   1 (7m27s ago)   10m
engine-image-ei-51cc7b9c-97ctd                      1/1     Running   0               11m
engine-image-ei-51cc7b9c-d8mrj                      1/1     Running   0               11m
instance-manager-d2da78f0ad5e1f5dd35d4375b9d24816   1/1     Running   0               10m
instance-manager-f484579a33af1d933c8617b0d7d0565b   1/1     Running   0               10m
longhorn-csi-plugin-jq85k                           3/3     Running   0               10m
longhorn-csi-plugin-kghjt                           3/3     Running   1 (7m8s ago)    10m
longhorn-driver-deployer-7d77d779dd-xtlhp           1/1     Running   0               11m
longhorn-manager-7v5nn                              2/2     Running   0               11m
longhorn-manager-w7v48                              2/2     Running   0               11m
longhorn-ui-7c784d8587-crfxc                        1/1     Running   0               11m
longhorn-ui-7c784d8587-hbdsv                        1/1     Running   0               11m
```

## Longhorn UI

Check ExternalIP of `longhorn-frontend`
```bash
$ k get svc -n longhorn-system
NAME                          TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)        AGE
longhorn-admission-webhook    ClusterIP      10.233.55.108   <none>          9502/TCP       26m
longhorn-backend              ClusterIP      10.233.46.117   <none>          9500/TCP       26m
longhorn-conversion-webhook   ClusterIP      10.233.2.243    <none>          9501/TCP       26m
longhorn-frontend             LoadBalancer   10.233.38.27    192.168.1.241   80:31661/TCP   26m
longhorn-recovery-backend     ClusterIP      10.233.46.81    <none>          9503/TCP       26m
```

## Test

```bash
k apply -f nginx.yaml
```

```bash
$ k get po
NAME                               READY   STATUS    RESTARTS   AGE
nginx-deployment-bf98589cb-7btcd   1/1     Running   0          47s
$ k get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM              STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
pvc-6bdf08a8-ebcf-46f7-b54a-ff06f24fc556   1Gi        RWO            Delete           Bound    default/test-pvc   longhorn       <unset>                          51s
$ k get pvc
NAME       STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
test-pvc   Bound    pvc-6bdf08a8-ebcf-46f7-b54a-ff06f24fc556   1Gi        RWO            longhorn       <unset>                 56s
```

PV will be removed automatically by deleting PVC.

