# ingress-nginx

## sample install

```bash
# install test sample
k apply -f fruit.yaml
# install ingress-nginx
k apply -f install-ingress-nginx.yml
# install ingress for path
k apply -f ingress-path.yml
# install ingress for host
k apply -f ingress-host.yml
# install ingress for tls(https)
k apply -f ingress-tls.yml
```
