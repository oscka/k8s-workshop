# nks alb-ingress-controller

ncp kubernetes service상에서 테스트를 위한 workshop

## install and test

아래와 같이 테스트를 위한 pod들과 alb-ingress-controller를 설치한다.

```bash
# install test sample
k apply -f nks-sample-services.yml
# install nks-alb
# kubectl apply -f https://raw.githubusercontent.com/NaverCloudPlatform/nks-alb-ingress-controller/main/docs/install/pub/install.yaml 과 동일
k apply -f install-nks-alb-ingress-controller.yml
```
아래와 같이 ingress를 생성하면 자동으로 ncp상에 LoadBalancer가 생성되며 서비스가 노출된다.

```
# install sample ingress
k apply -f sample-alb-ingress.yml
```

좀 더 자세한 내용은 [이곳](https://osc-korea.atlassian.net/wiki/spaces/consulting/pages/685211651/nks+alb+ingress+controller)을 참고한다. 


```
# simple-api deployment를 위한 secret생성
kubectl create secret docker-registry dockersecret --docker-username="oscka" \
--docker-password="1P9V4X#yn#6G" --docker-server=https://index.docker.io/v1/

```
