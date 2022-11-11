# k8s-workshop


## devops

### 소개

kubernetes 클러스터를 기반으로 CI,CD 환경을 구성하는 hands-on workshop 입니다.

다음과 같은 구성으로 되어 있습니다.

- k3s
- ingress-nginx
- jenkins (클러스터 외부에 설치)
- argocd
- mysql
- sample-api
- sample-fe
- github(외부 서비스)
- docker hub

### 설치

msa-starter-kit을 통해 다음과 같이 설치

- k3s, ingress-nginx, argocd, mysql 은 클러스터상에 설치
- jenkins는 클러스터 밖에 별도로 설치


```bash
./run-play.sh  "tool-basic, ohmyzsh, helm-repo, k3s, ingress-nginx, jenkins, argocd, mysql"
```

설치하면 아래와 같은 구성요소들이 설치됩니다.


구성도 첨부


### 구성









