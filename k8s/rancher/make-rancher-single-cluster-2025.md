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

### 2-7. NFS 클라이언트 설치
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

