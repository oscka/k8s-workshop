# ingress-nginx

nginx ingress controller 테스트를 위한 workshop

## install and test

아래와 같이 테스트 대상 fruit pod들과 ingress-nginx 컨트롤러를 설치한다.

```zsh
# install test sample
k apply -f fruit.yaml
# install ingress-nginx
# kubectl apply -f <https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.4.0/deploy/static/provider/baremetal/deploy.yaml 와 동일
k apply -f install-ingress-nginx.yml
```

상황에 맞게 ingress를 생성하여 테스트 한다. 하단의 세가지 방법은 각각 테스트 후 지워야 다른 case를 테스트 할 수 있다. ingress-tls의 경우 테스트를 위해 인증서 파일이 필요하다.

```zsh
# install ingress for path
k apply -f ingress-path.yml
# install ingress for host
k apply -f ingress-host.yml
# install ingress for tls(https)
k apply -f ingress-tls.yml
```

좀 더 자세한 내용은 [이곳](https://osc-korea.atlassian.net/wiki/spaces/consulting/pages/664305665/Nginx+Ingress+Controller)을 참고한다.