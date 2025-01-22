# wg-easy

wg-easy for baremetal kubernetes

## Preparation

- [MetalLB is set up](../metallb)
- [Longhorn is set up](../longhorn)

## Deployment

```
k apply -f wg-easy.yaml
```

## Client Registration

access `wg-easy-frontend` external IP by browser

## Router

Configure your router to forward 51820/udp to `wg-svc`'s external IP

## Reference
https://github.com/wg-easy/wg-easy/wiki/Using-WireGuard-Easy-with-Kubernetes

