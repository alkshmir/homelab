## Deployment

1. `k create ns homelab-dns`
1. `k apply -f etcd.yaml -f dns.yaml`
   Note: corefile is now only configured to use etcd for `homelab.local` domains.
1. Confrim coredns is working: `dig @192.168.1.240 google.com`
1. Deploy external-dns to synchronize ingress A record
   ```
   k apply -f external-dns.yaml
   ```

## Ingress Example

If your ingress is configured as `<your-service>.homelab.local`, run following command to confirm the synchronization of ingress record.

```bash
curl --dns-servers 192.168.1.240 http://<your-service>.homelab.local
```

If `--dns-servers` option is not supported in your `curl`, try
```bash
docker run --rm alpine/curl --dns-servers 192.168.1.240 http://<your-service>.homelab.local
```

If you want to access ingress service with your client PC, configure the target DNS server to use coreDNS loadBalancerIP (`192.168.1.240` in this case).
