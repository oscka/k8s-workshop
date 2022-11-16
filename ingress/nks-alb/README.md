# nks alb-ingress-controller

ncp kubernetes service상에서 단순한 서비스를 배포하고 ALB를 통해 노출하기 위한 workshop

## sample1. fruit deploy and test

sanple 1은 다음 ncp의 공식 산출물 내용을 기반으로 한다.

- [ncp alb ingress controller 설정](https://guide.ncloud-docs.com/docs/k8s-k8suse-albingress)
- [ncp alb ingress controller 활용예제](https://guide.ncloud-docs.com/docs/k8s-k8sexamples-albingress)

아래와 같이 테스트를 위한 deployment, service들과 alb-ingress-controller를 설치한다.

```bash
# install test sample
kubectl apply -f nks-sample-services.yml

# install nks-alb-ingress-controller
# kubectl apply -f https://raw.githubusercontent.com/NaverCloudPlatform/nks-alb-ingress-controller/main/docs/install/pub/install.yaml 과 동일
kubectl apply -f install-nks-alb-ingress-controller.yml
```
아래와 같이 ingress를 생성하면 자동으로 ncp상에 LoadBalancer가 생성되며 서비스가 노출된다.

```zsh
# fruit ingress 적용
kubectl apply -f nks-sample-alb-ingress.yml
```

좀 더 자세한 내용은 [이곳](https://osc-korea.atlassian.net/wiki/spaces/consulting/pages/685211651/nks+alb+ingress+controller)을 참고한다. 

## sample2. simple-api deploy and test

sample2 에서는 springboot 기반의 simple-api를 deploy하고 test하는 절차에 관하여 설명한다.

```zsh
# simple-api deployment, service 생성
kubectl apply -f simple-api-deployment.yaml
# simple-api alb ingress 생성
kubectl apply -f simple-api-alb-ingress.yaml
```

