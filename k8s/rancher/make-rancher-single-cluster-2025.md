본 가이드는 Rancher 기반 클러스터에 여러가지 오픈소스 및 어플리케이션을 설치 및 배포하는 과정을 설명합니다.

- [0. 개요](#0-개요)
- [1. 사전작업](#1-사전작업)
- [2. rke2 설치](#2-rke2-설치)
- [3. OSS설치](#1-사전작업)
- [4. 어플리케이션 배포하기](#4-어플리케이션-배포하기)


## 0. 개요

rancher 기반의 single cluster 설치 가이드(2025년 버전)

- 기존 가이드는 3년전 작성된 버전 기준이므로, 패치 등의 현행화를 추가 반영.
- ubuntu 22.04 버전을 기준으로 설치를 수행하였으며 반드시 server 배포판 안의 minimal 버전으로 설치하지 말것.
(ingress controller가 정상 구동되지 않음)
- 하나의 서버에서 구동 테스트 수행, ubuntu 22.04, kvm기반의 VM

## 1. 사전작업
```
-- 사전작업
swapoff -a
apt-get upgrade -y
apt-get dist-upgrade -y
apt-get update -y
systemctl stop ufw && ufw disable && iptables -F
```

## 2. RKE2 설치
```
-- RKE2 설치
curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE="server" sh -
systemctl enable rke2-server.service
systemctl start rke2-server.service
```

### 2-1. kubeconfig 관련 설정 추가
```
-- kubectl 관련 설정 추가
mkdir ~/.kube/
cp /etc/rancher/rke2/rke2.yaml ~/.kube/config
export PATH=$PATH:/var/lib/rancher/rke2/bin/
echo 'export PATH=/usr/local/bin:/var/lib/rancher/rke2/bin:$PATH' >> ~/.bashrc
echo 'source <(kubectl completion bash)' >>~/.bashrc
echo 'alias k=kubectl' >>~/.bashrc
echo 'complete -F __start_kubectl k' >>~/.bashrc
```


### 2-2. k9s 설치
```
-- k9s설치
wget https://github.com/derailed/k9s/releases/download/v0.50.6/k9s_linux_amd64.deb
dpkg -i k9s_linux_amd64.deb
```



### 2-3. helm 설치

```
-- helm 설치
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

### 2-4. cert-manager 설치

```
-- cert-manager 설치
helm repo add jetstack https://charts.jetstack.io
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.17.2/cert-manager.yaml
```


### 2-5. rancher 설치

```
-- rancher(rancher-ui, rancher-dashboard라고도 함)
-- 사전에 cert-manager가 설치되어 있어야 함
kubectl create namespace cattle-system
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
helm repo update
helm search repo rancher-stable
helm install rancher rancher-stable/rancher --namespace cattle-system --set hostname=rancher.192.168.122.37.nip.io --set replicas=1
```

rancher desktop 비번은 다음과 같이 설정함 - osckorea1234!!


### 2-6. NFS 서버 설치

```
apt install nfs-common nfs-kernel-server rpcbind

mkdir /share
chmod 707 /share
cd /share;touch aaa bbb ccc

vi /etc/exports
/share 192.168.122.* (rw, root_squash,sync) --> 이렇게 하면 연동이 안되어서 아래와 같이 입력
/share *(rw,sync,no_subtree_check,no_root_squash)

systemctl enable nfs-server
systemctl restart nfs-server
systemctl status nfs-server
```

### 2-7. NFS 클라이언트 설치(필요시)
이 부분은 NFS 서버에 연결하여 로컬에 공유 스토리지를 마운트 하기 위한 방법이며, 클러스터의 스토리지 클래스로 설정하기 위해ㅔ서는 nfs-common만 설치되어 있으면 된다.
```
apt install nfs-common

mkdir /myshare
showmount -e 192.168.122.37
mount -t 192.168.122.37:/share myshare

vi /etc/fstab *
192.168.122.37:/share /root/myshare nfs default 0 0
```

### 2-8. k8s storageClass 설정

```
nfs_server="192.168.122.37"
nfs_dir_path="/share"

kubectl create namespace nfs-provisioner
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
helm repo update

helm install -n nfs-provisioner nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
--set nfs.server=$nfs_server \
--set nfs.path=$nfs_dir_path \
--set storageClass.defaultClass=true  # 맨마지막 default설정하지않으려면 지워도됨
```


```
# test-claim.yaml

kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: test-claim
spec:
  storageClassName: nfs-client
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Mi
```
위 yaml을 생성하여 PVC, PV가 자동으로 생성되는지 확인 




## 3. OSS 설치

### 3-1.gitlab 설치

cert-manager를 통해 내부적으로 인증서를 관리할 수 있으나, 여기서는 on prem에서 사설인증서를 적용하기 위해 설치에서 제외한다.

```
# 1. 바로 설치
kubectl create ns cicd
helm repo add gitlab https://charts.gitlab.io/
helm repo update
helm upgrade -n cicd --install gitlab gitlab/gitlab \
  --timeout 600s \
  --set global.hosts.domain=k2p-option.com \
  --set global.hosts.externalIP=192.168.122.37 \
  --set certmanager-issuer.email=me@example.com \
  --set certmanager.install=false \
  --set global.edition=ce:PVC

#초기 비밀번호 확인(root계정)
kubectl get secret -n cicd gitlab-gitlab-initial-root-password -o jsonpath="{.data.password}" | base64 -d ; echo

# 2. 차트 다운로드 후 설치
helm search repo -l gitlab/gitlab
helm pull gitlab/gitlab --version 8.9.2
tar -xzvf gitlab-8.9.2.tgz # 압축 해제

# 2-1. values.yaml 내용 수정 - 다음과 같은 설정들을 수정한다.
#-postgresql 버전 변경
#-certmanager관련 설치옵션 비활성화
#-prometheus 설치옵션 비활성화

# 2-2. 설치
helm install gitlab gitlab/gitlab -f values.yaml -n cicd
```


### 3-2. opensearch 설치(작성중)

https://www.instaclustr.com/education/opensearch/getting-started-with-opensearch-2-quick-tutorials/
helm repo add opensearch https://opensearch-project.github.io/helm-charts/
helm repo update


### 3-3. Kafka 설치

```zsh
helm repo add kafka-repo https://charts.bitnami.com/bitnami
helm repo update
helm pull kafka-repo/kafka
```

위와 같이 받은 저장소의 values.yaml 파일을 ../k8s/kafka/values.yaml 파일로 대체한 뒤 설치한다.

```zsh
helm install my-kafka kafka-repo/kafka -f values.yaml -n kafka
```


## 4. Multi Node

### 4.1. 준비
클러스터 설정을 위한 멀티 노드 환경 세팅
worker 노드로 사용할 vm 생성 (원하는 개수 만큼)
```zsh
# 1. 새로 생성한 각 vm에서 OS 업데이트 및 방화벽 해제 설정 (master 노드와 동일)
swapoff -a
apt-get upgrade -y
apt-get dist-upgrade -y
apt-get update -y
systemctl stop ufw && ufw disable && iptables -F
```
```zsh
# 2. RKE2 agent 설치 (master 노드와 다름!! agent)
curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE="agent" sh -
systemctl enable rke2-agent.service
```

```zsh
# 3. master node 토큰 확인 (master 노드에서 수행)
cat /var/lib/rancher/rke2/server/node-token

# 샘플
# K10bb3a06960ea16375ed144cd8f5a8a98e7a5fc629b675021fa9447fc5daba8353::server:2f17234499ff71cd092f4d442fe900fb
```

```zsh
# 4. master node config.yaml 파일 생성
vi /etc/rancher/rke2/config.yaml

--- config.yaml
advertise-address: 마스터노드IP
token: master node 토큰
--- 

--- 샘플 config.yaml
advertise-address: 192.168.0.101
token: K104e9f8cb3257d84fbf1eeb633ca413f8126d4908eb245168b0259a03c30e54c8f::server:1e0ce06443d615b51fc7be3114fadc1e
--- 
```
마스터 노드 설정이 없을 경우 워커 노드에 설정한 ip(예:192.168.0.101)가 아닌

내부 ip(예:10.0.2.15)로 연결시 거부되어 구동되지 않음

- rke2-canal

```zsh
# 5. worker node config.yaml 파일 생성
mkdir -p /etc/rancher/rke2/
vi /etc/rancher/rke2/config.yaml

--- config.yaml
server: https://마스터노드IP:9345
token: master node 토큰
--- 
--- 샘플 config.yaml
server: https://192.168.0.101:9345
token: K10bb3a06960ea16375ed144cd8f5a8a98e7a5fc629b675021fa9447fc5daba8353::server:2f17234499ff71cd092f4d442fe900fb
---
```
여기까지 진행하면 worker 노드 설정은 끝 \
vm 계정 권한 이슈가 발생할 수 있으므로 테스트 진행을 위한 환경에서는 root 권한 사용 권장 


### 4.2. Redis(OSS) 설치
helm이 아닌 k8s manifest를 이용하여 생성한다.

redis 클러스터 설치(../redis 경로 참고)

```zsh
kubectl create -f redis-cluster-master.yaml
kubectl create -f redis-cluster-slave.yaml
kubectl create -f redis-cluster-svc.yaml
```

redis 클러스터 생성

```zsh
$ redis-cli --cluster create --cluster-replicas 1 192.168.122.102:6379 192.168.122.101:6379 192.168.122.100:6379 192.168.122.102:6380 192.168.122.101:6380 192.168.122.100:6380
>>> Performing hash slots allocation on 6 nodes...
Master[0] -> Slots 0 - 5460
Master[1] -> Slots 5461 - 10922
Master[2] -> Slots 10923 - 16383
Adding replica 192.168.122.101:6380 to 192.168.122.102:6379
Adding replica 192.168.122.100:6380 to 192.168.122.101:6379
Adding replica 192.168.122.102:6380 to 192.168.122.100:6379
M: 8b35ccb1193b0b5e4634ee1510442373dd2f49d5 192.168.122.102:6379
   slots:[0-5460] (5461 slots) master
M: f1ccbebaa3bf7657bfc4aa46f4213bd307600740 192.168.122.101:6379
   slots:[5461-10922] (5462 slots) master
M: 957fc5a1a5ca64478337a7cf2ca057113ac845a7 192.168.122.100:6379
   slots:[10923-16383] (5461 slots) master
S: 31882c72ba0184da25b81bf0f5eda11fb35ef6f8 192.168.122.102:6380
   replicates 957fc5a1a5ca64478337a7cf2ca057113ac845a7
S: a76603262c46f101d5afbb7b4b37b18c766d7bba 192.168.122.101:6380
   replicates 8b35ccb1193b0b5e4634ee1510442373dd2f49d5
S: 04698386611798d00b1fb9ec33bede9629c672d6 192.168.122.100:6380
   replicates f1ccbebaa3bf7657bfc4aa46f4213bd307600740
Can I set the above configuration? (type 'yes' to accept): yes
>>> Nodes configuration updated
>>> Assign a different config epoch to each node
>>> Sending CLUSTER MEET messages to join the cluster
Waiting for the cluster to join

>>> Performing Cluster Check (using node 192.168.122.102:6379)
M: 8b35ccb1193b0b5e4634ee1510442373dd2f49d5 192.168.122.102:6379
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
S: a76603262c46f101d5afbb7b4b37b18c766d7bba 192.168.122.101:6380
   slots: (0 slots) slave
   replicates 8b35ccb1193b0b5e4634ee1510442373dd2f49d5
S: 31882c72ba0184da25b81bf0f5eda11fb35ef6f8 192.168.122.102:6380
   slots: (0 slots) slave
   replicates 957fc5a1a5ca64478337a7cf2ca057113ac845a7
M: 957fc5a1a5ca64478337a7cf2ca057113ac845a7 192.168.122.100:6379
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
S: 04698386611798d00b1fb9ec33bede9629c672d6 192.168.122.100:6380
   slots: (0 slots) slave
   replicates f1ccbebaa3bf7657bfc4aa46f4213bd307600740
M: f1ccbebaa3bf7657bfc4aa46f4213bd307600740 192.168.122.101:6379
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```

redis 클러스터 사용

```zsh
# 노드 상태 확인
$ redis-cli cluster nodes
a76603262c46f101d5afbb7b4b37b18c766d7bba 192.168.122.101:6380@16380 slave 8b35ccb1193b0b5e4634ee1510442373dd2f49d5 0 1749099040374 1 connected
8b35ccb1193b0b5e4634ee1510442373dd2f49d5 192.168.122.102:6379@16379 myself,master - 0 1749099040000 1 connected 0-5460
31882c72ba0184da25b81bf0f5eda11fb35ef6f8 192.168.122.102:6380@16380 slave 957fc5a1a5ca64478337a7cf2ca057113ac845a7 0 1749099040575 3 connected
957fc5a1a5ca64478337a7cf2ca057113ac845a7 192.168.122.100:6379@16379 master - 0 1749099039971 3 connected 10923-16383
04698386611798d00b1fb9ec33bede9629c672d6 192.168.122.100:6380@16380 slave f1ccbebaa3bf7657bfc4aa46f4213bd307600740 0 1749099040173 2 connected
f1ccbebaa3bf7657bfc4aa46f4213bd307600740 192.168.122.101:6379@16379 master - 0 1749099040000 2 connected 5461-10922

# 클러스터 정보 조회
redis-cli cluster info
# 접속 및 테스트
redis-cli -c
/data # redis-cli -c
127.0.0.1:6379> set hello world
OK
127.0.0.1:6379> get hello
"world"
```




## 5. 어플리케이션 배포하기

simple-api 어플리케이션 배포 

https://github.com/oscka/simple-gitops/tree/main/simple-api/rolling-update-no-istio 하위의 다음 파일을 대상으로 수행

- simple-api-deployment.yaml
- simple-api-ingress.yaml
- simple-api-svc.yaml

```zsh
#simple-api-deployment.yaml 파일 안의 내용을 수정
...
image: oscka/simple-api ->  image: oscka/simple-api:0.0.1
...
#simple-api-ingress.yaml 파일 안의 내용을 수정(external ip주소에 맞게)
  - host: "simple-api.192.168.56.10.sslip.io"
# 디버깅을 위해 임시 인스턴스를 생성하여 해당 인스턴스 안에서 curl명령으로 확인 가능
kubectl run dev-tools -it --image oscka/osc-devtools -n sample 
# 다음과 같이 배포를 수행합니다.
kubectl apply -f ./simple-api-deployment.yaml
# 호출하면서 로그를 확인합니다.
curl http://simple-api.192.168.122.37.sslip.io/api/simple
```




