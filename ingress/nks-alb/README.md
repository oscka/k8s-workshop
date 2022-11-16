# nks alb-ingress-controller

ncp kubernetes service상에서 단순한 서비스를 배포하고 ALB를 통해 노출하기 위한 workshop

## sample1. fruit deploy and test

sanple 1은 다음 ncp의 공식 산출물 내용을 기반으로 한다.

- [ncp alb ingress controller 설정](https://guide.ncloud-docs.com/docs/k8s-k8suse-albingress)
- [ncp alb ingress controller 활용예제](https://guide.ncloud-docs.com/docs/k8s-k8sexamples-albingress)

아래와 같이 테스트를 위한 deployment, service들과 alb-ingress-controller를 설치한다.

```zsh
# fruit deployment, service 생성
kubectl apply -f nks-sample-services.yml

# nks alb ingress controller 설치
# kubectl apply -f https://raw.githubusercontent.com/NaverCloudPlatform/nks-alb-ingress-controller/main/docs/install/pub/install.yaml 과 동일
kubectl apply -f install-nks-alb-ingress-controller.yml
```
아래와 같이 ingress를 생성하면 자동으로 ncp상에 LoadBalancer가 생성되며 서비스가 노출된다.

```zsh
# fruit ingress 적용
kubectl apply -f nks-sample-alb-ingress.yml
```

노출되는 주소를 호출하여 테스트한다. 아래는 그 예이다.

```zsh
# ingress 정보를 확인
$ k get ing
# 해당 ALB주소를 호출하여 테스트(예)
$ curl http://ing-default-samplealbing-83129-13830022-f736d2e6302c.kr.lb.naverncp.com/naver
Server address: 198.18.0.167:80
Server name: naver-6f4d9bb7fc-ttbq2
Date: 02/Nov/2022:06:58:53 +0000
URI: /naver
Request ID: cd18737c98f1c13c5e813b4a16a22a1f
$ curl http://ing-default-samplealbing-83129-13830022-f736d2e6302c.kr.lb.naverncp.com/navercloud
Server address: 198.18.0.167:80
Server name: naver-6f4d9bb7fc-ttbq2
Date: 02/Nov/2022:07:06:56 +0000
URI: /navercloud
Request ID: b5d96c8073e2ccee8883c97424312287
$ curl http://ing-default-samplealbing-83129-13830022-f736d2e6302c.kr.lb.naverncp.com/platform
Server address: 198.18.0.194:80
Server name: platform-785f88c577-gww2t
Date: 02/Nov/2022:07:07:20 +0000
URI: /platform
Request ID: f6454d6176696665ce96a3becd404a21
```

좀 더 자세한 내용은 [이곳](https://osc-korea.atlassian.net/wiki/spaces/consulting/pages/685211651/nks+alb+ingress+controller)을 참고한다. 

## sample2. simple-api deploy and test

sample2 에서는 springboot 기반의 simple-api를 deploy하고 test하는 절차에 관하여 설명한다.

[simple-api](https://github.com/oscka/simple-api)

사전에 위의 sample-api 프로젝트를 빌드하고 containerize 하여 docker hub에 push하였고, 그 이후 클러스터에 배포하기 위해
아래와 같이 수행한다.

```zsh
# simple-api deployment, service 생성
kubectl apply -f simple-api-deployment.yaml
# simple-api alb ingress 생성
kubectl apply -f simple-api-alb-ingress.yaml
```

노출되는 주소를 호출하여 테스트한다. 아래는 그 예이다.

```zsh
# ingress 정보를 확인
$ k get ing
# 해당 ALB주소를 호출하여 테스트(예)
$ curl ing-default-simplealbing-49ca1-14424922-5a6fdeca4ce0.kr.lb.naverncp.com/api/simple
```

simple-api-deployment의 경우 private registry에 접근하기 위해 secret을 k8s에 생성하여야 하며 다음과 같이 생성하고 해당 secret의 이름을 deployment에 적용한다.

```zsh
kubectl create secret docker-registry dockersecret --docker-username="{사용자ID}" \
--docker-password="{사용자PW}" --docker-server=https://index.docker.io/v1/
```
