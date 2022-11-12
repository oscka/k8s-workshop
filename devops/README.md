# k8s-workshop

## devops

### 소개

kubernetes 클러스터를 기반으로 CI,CD 환경을 구성하는 hands-on workshop

다음과 같은 구성으로 되어 있다.

- k3s
- ingress-nginx
- jenkins (클러스터 외부에 설치)
- argocd
- mysql
- sample-api
- sample-fe
- github(외부 서비스)
- docker hub(외부 서비스)

### 설치

msa-starter-kit을 통해 다음과 같이 설치

- k3s, ingress-nginx, argocd, mysql 은 클러스터상에 설치
- jenkins는 클러스터 밖에 별도로 설치


```bash
./run-play.sh  "tool-basic, ohmyzsh, helm-repo, k3s, ingress-nginx, jenkins, argocd, mysql"
```

설치하면 아래와 같은 구성요소들이 설치된다.

![cicd-msa-env](https://user-images.githubusercontent.com/112376183/201487394-ebf3a507-aa51-4cb1-87e3-08b283a868fe.png)


### CI구성

Jenkins의 경우 job실행 속도 문제로 클러스터 밖의 환경에 별도로 설치하도록 구성되어 있으며 설치만 수행하고 있음
이후 아래와 같은 작업이 필요하다.

```bash
#1.jenkins계정에 docker 실행권한 부여(재시작,재로그인 후 반영)
sudo usermod -aG docker jenkins
sudo service docker restart

#2.계정 생성 및 비밀번호 변경
#admin/admin1234 로 관리자 계정 생성(UI에서)
cat /var/lib/jenkins/secrets/initialAdminPassword

#3.로그인 후 플러그인 설치
# 플러그인 3개 - git parameter, workspace cleanup, docker pipeline

#4.secret 생성 - git-credential, imageRegistry-credential
#jenkins관리 - credential상에 위의 이름으로 생성하고 각각 github의 accesstoken 정보와 docker hub의 ID/PW정보를 입력해 둔다.

#5.job 생성
#- jekins UI상에서 sample-api-build를 pipeline job으로 생성
#- git address - https://github.com/oscka/sample-api.git
#- 이 빌드는 매개변수가 있습니다. 를 선택하여 TAG를 입력
#- pipeline script는 SCM에서 가져온 Jenkinsfile을 선택

#6. 빌드 도구 설치
# skaffold, kustomize 설치 - ./run-play.sh  "skaffold, kustomize"
# jenkins계정에서 docker 수행이 가능해야 함
```

### CD구성

CI가 완료되면 1.MSA어플리케이션은 이미지 형태로 containerize되어 docker hub에 deploy된다. 또한 동시에 2. gitops repository의 해당 어플리케이션 버전을 업데이트한다.

```zsh
#1. argocd에 로그인하여 setting > repository 메뉴에서 새로운 repository를 등록한다.
# gitops address - https://github.com/oscka/sample-gitops.git
# https방식으로 연결시 github id와 access token이 필요하다.(password는 보안때문에 사용 불가)

#2. 배포를 위한 app을 등록한다.
# Application Name - sample-api
# Project Name - default
# Repository URL - https://github.com/oscka/sample-gitops.git(자동입력 선택)
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



