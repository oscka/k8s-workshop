# k8s-workshop

- [k8s-workshop](#k8s-workshop)
  - [devops](#devops)
    - [소개](#소개)
    - [설치](#설치)
    - [CI구성](#ci구성)
    - [CD구성](#cd구성)
    - [테스트](#테스트)

## devops

### 소개

본 문서는 kubernetes 클러스터를 기반으로 CI,CD 환경을 구성하는 hands-on workshop이다.

다음과 같은 구성으로 되어 있다.

- k3s
- ingress-nginx
- jenkins (클러스터 외부에 설치)
- argocd
- mysql
- sample-api
- github(외부 서비스)
- docker hub(외부 서비스)

### 환경준비

다음과 같이 vagrant(virtualbox)기반으로 vm을 두 대 생성한다. 
- vm1 - agent역할, ansible 코드를 받아 target서버에 설치를 수행한다.
- vm2 - target역할, jenkins 및 k3s기반 클러스터, sample-api가 실행된다.
vm1은 로컬이 리눅스 환경이거나, 윈도우의 wsl이라도 상관은 없다.

각 vm을 띄우는 방법은 [msa-starter-kit](https://github.com/oscka/msa-starter-kit/tree/develop/test-vm1)의 Vagrantfile을 참고한다. 로컬 - vm1 - vm2간의 ssh 및 http통신이 원활하여야 한다.

### 설치

[msa-starter-kit](https://github.com/oscka/msa-starter-kit)을 통해 다음과 같이 설치

- k3s, ingress-nginx, argocd, mysql 은 클러스터상에 설치
- jenkins는 클러스터 밖에 별도로 설치


```bash
./run-play.sh  "tool-basic, ohmyzsh, helm-repo, k3s, ingress-nginx, jenkins, docker, argocd, mysql"
./run-play.sh  "skaffold, kustomize"
```

설치하면 아래와 같은 구성과 같이 설치된다.
- 파란색 흐름 - CI(통합빌드)를 의미, 소스코드를 받아 빌드, dockerizing하고 docker hub에 push한다.
- 빨간색 흐름 - CD(배포)를 의미, Gitops에 변경된 버전을 확인하여 정의된 배포 전략에 따라 배포를 수행한다.

![cicd-msa-env](https://user-images.githubusercontent.com/112376183/201487394-ebf3a507-aa51-4cb1-87e3-08b283a868fe.png)

다음 프로젝트를 fork하여 각자의 계정에 프로젝트를 생성한다.

- sample-api - https://github.com/oscka/sample-api.git
- sample-gitops - https://github.com/oscka/sample-gitops.git

향후 진행할 샘플에서 연결할 프로젝트는 각자 개인계정의 다음 프로젝트를 기반으로 한다.

- sample-api - https://github.com/{{개인ID}}/sample-api.git
- sample-gitops - https://github.com/{{개인ID}}/sample-gitops.git


### CI구성

Jenkins의 경우 job실행 속도 문제로 클러스터 밖의 환경에 별도로 설치하도록 구성되어 있으며 설치만 수행하고 있음
이후 아래와 같은 작업이 필요하다.

```bash
#1.containerize를 위하여 jenkins계정에 docker 실행권한 부여(docker daemon 재시작,재로그인 후 반영)
sudo usermod -aG docker jenkins
sudo service docker restart

#2.계정 생성 및 비밀번호 변경
#admin/admin1234 로 관리자 계정 생성(UI에서)
cat /var/lib/jenkins/secrets/initialAdminPassword

#3.로그인 후 플러그인 설치
# 플러그인 3개 - git parameter, workspace cleanup, docker pipeline
# docker pipeline의 경우 초기세팅시 선택 불가하므로 차후 jenkins관리메뉴에서 설치할 것
# 플러그인 설치가 안될 경우 jenkins재시작하여 다시 시도 

#4.secret 생성 - git-credential, imageRegistry-credential
#jenkins관리 > credential상에 위의 이름으로 생성하고 각각 github의 accesstoken 정보와 docker hub의 ID/PW정보를 입력해 둔다.
#github의 access token은 각자의 계정에서 Account > Setting > Developer setting에서 classic token으로 생성하며, Repo관련 권한을 전부 부여
#docker hub계정은 공용으로 별도로 전달한 것을 사용한다.(개인계정을 써도 상관 없음)

#5.job 생성
#- jekins UI상에서 sample-api-build를 pipeline job으로 생성
#- 상단에서 "이 빌드는 매개변수가 있습니다." 를 선택하여 TAG를 입력
#- 하단 Pipeline 메뉴에서  "Pipeline Script from SCM"을 선택하고
#- git address(https://github.com/{{개인ID}}/sample-api.git)와 브랜치 입력
#- pipeline script는 SCM에서 가져온 Jenkinsfile을 선택

#6. 빌드 도구 설치
# skaffold, kustomize 설치 - ./run-play.sh  "skaffold, kustomize"
# jenkins계정에서 docker 수행이 가능해야 함
```

### CD구성

CI가 완료되면 1.MSA어플리케이션은 이미지 형태로 containerize되어 docker hub에 deploy된다. 또한 동시에 2. gitops repository의 해당 어플리케이션 버전을 업데이트한다.

```zsh
#1. argocd에 로그인하여 setting > repository 메뉴에서 새로운 repository를 등록한다.
# gitops address - https://github.com/{{개인ID}}/sample-gitops.git
# https방식으로 연결시 github id와 access token이 필요하다.(password는 보안때문에 사용 불가)

#2. 배포를 위한 app을 등록한다.
# Application Name - sample-api
# Project Name - default
# Repository URL - https://github.com/{{개인ID}}/sample-gitops.git(자동입력 선택)
# Revision - main(자동입력 선택)
# Path - sample-api/rolling-update-no-istio(자동입력 선택)
# Cluster URL - https://kubernetes.default.svc(자동입력 선택)
# Namespace - api
# 맨 아래부분에는 Kustomize를 통해 기 입력된 항목들이 조회된다.
# Create 후 Refresh, SYNC 버튼을 한 번씩 눌러주면 gitops의 내용이 클러스터에 deploy된다.
```

위와 같이 수행하여 application이 pod로 정상적으로 cluster에 구동되었음을 확인한다.
이후 다음과 같이 버전을 올려 테스트 해 본다.

```zsh
#3. 어플리케이션의 버전을 올린 뒤 tag의 버전을 수정한다
# 버전을 올리기 위해 소스코드를 수정한다.
# git add ./*;git commit -m "version up";git push
# git tag 0.0.19;git push

#4. jenkins에서 해당job을 찾아 빌드를 수행한다.
#5. argocd에서 sync하여 버전이 제대로 적용되었는지 확인한다.
```


### 테스트

생성한 application 테스트를 위해 다음과 같은 k8s manifest를 저장하여 클러스터에 적용한다.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: sample-api-ingress
  namespace: api
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
  - host: "sample-api.{{서버의IP주소}}.sslip.io"
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: sample-api-svc
            port:
              number: 8080
```

다음과 같이 생성한다. 

k apply -f ./sample-api-ingress.yml -n api 

생성된 다음 주소를 호출하고 테스트한다. 

http://sample-api.{{서버의IP주소}}.sslip.io/swagger-ui.html



