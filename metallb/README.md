# MetalLB

## Deploy

```
helm repo add metallb https://metallb.github.io/metallb
kubectl create namespace metallb-system
helm install metallb metallb/metallb -n metallb-system
```

## IP Address Pool

```
kubectl apply -f ipaddresspool.yaml
```

## Test

```
kubectl apply -f nginx.yaml
```

You will have LoadBalancer service with external IP.

```
$ k get svc
NAME            TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)        AGE
kubernetes      ClusterIP      10.233.0.1      <none>          443/TCP        8h
nginx-service   LoadBalancer   10.233.47.193   192.168.1.241   80:32639/TCP   3s
```

