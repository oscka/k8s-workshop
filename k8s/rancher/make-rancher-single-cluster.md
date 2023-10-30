간단한 로컬 클러스터 테스트를 위한 rancher 설치 가이드(유이사님 원문을 수정)

## 스크립트 설치 및 서비스 시작

RKE2 provides an installation script that is a convenient way to install it as a service on systemd based systems. hostname 에 special character (_) 등이 있으면, RKE2 서버 구동/설치에 문제가 있을 수 있음

```java
swapoff -a  (/etc/fstab swap comment out)
apt-get upgrade -y
apt-get dist-upgrade -y
apt-get update -y
systemctl stop ufw && ufw disable && iptables -F
```

```java
# curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE="server" sh -
[INFO]  finding release for channel stable
[INFO]  using v1.21.5+rke2r2 as release
[INFO]  downloading checksums at https://github.com/rancher/rke2/releases/download/v1.21.5+rke2r2/sha256sum-amd64.txt
[INFO]  downloading tarball at https://github.com/rancher/rke2/releases/download/v1.21.5+rke2r2/rke2.linux-amd64.tar.gz
[INFO]  verifying tarball
[INFO]  unpacking tarball file to /usr/local
# systemctl enable rke2-server.service
```

※ SAN 설정여기서는 SAN 예제 값으로, `my-kubernetes-domain.com`, `another-kubernetes-domain.com` 를 추가하는 것으로 진행

```java
# vi /etc/rancher/rke2/config.yaml
tls-san:
  - my-kubernetes-domain.com
  - another-kubernetes-domain.com
```

rke2-server 서비스 시작

```java
systemctl start rke2-server.service
```

## 설치확인

“Node synced successfully” 메세지 나오면 설치 완료

```java
# systemctl status rke2-server.service
# journalctl -u rke2-server -f
Jul 07 08:39:16 rke2-1 rke2[4327]: I0707 08:39:16.178631    4327 event.go:291]
"Event occurred" object="rke2-1" kind="Node" apiVersion="v1" type="Normal" reason="Synced" message="Node synced successfully"
```

## kubeconfig 환경설정

```java
mkdir ~/.kube/
cp /etc/rancher/rke2/rke2.yaml ~/.kube/config
export PATH=$PATH:/var/lib/rancher/rke2/bin/
echo 'export PATH=/usr/local/bin:/var/lib/rancher/rke2/bin:$PATH' >> ~/.bashrc
echo 'source <(kubectl completion bash)' >>~/.bashrc
echo 'alias k=kubectl' >>~/.bashrc
echo 'complete -F __start_kubectl k' >>~/.bashrc
```

## Node 확인

control plane, etcd, master 역할의 노드 1개 확인 가능

Note that the `rke2 server` process listens on port `9345` for new nodes to register

```java
# kubectl get nodes
NAME   STATUS   ROLES                       AGE   VERSION
rke2   Ready    control-plane,etcd,master   15m   v1.20.7+rke2r1
```
Ingress 확인

```java
# kubectl get ingress -A
NAMESPACE       NAME      CLASS    HOSTS                  ADDRESS   PORTS     AGE
cattle-system   rancher   <none>   192.168.0.221.nip.io             80, 443   30m

# kubectl get ds -n kube-system
NAME                            DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
kube-proxy                      1         1         1       1            1           kubernetes.io/os=linux   4m36s
rke2-canal                      1         1         1       1            1           kubernetes.io/os=linux   4m36s
rke2-ingress-nginx-controller   1         1         1       1            1           kubernetes.io/os=linux   3m23s
```

## 그밖의도구 설치

### Cert-Manager 설치

```java
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.5.4/cert-manager.yaml
kubectl -n cert-manager rollout status deploy/cert-manager
kubectl -n cert-manager rollout status deploy/cert-manager-webhook
kubectl get pods --namespace cert-manager
```

### HELM 설치

```java
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

### Rancher UI 배포 ( cattle-system )

여기서는, Rancher 서버 replicas=1 로 배포, 노드가 3개일 경우, 3개로 변경

```java
kubectl create namespace cattle-system
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
helm repo update
helm search repo rancher-stable
helm install rancher rancher-stable/rancher --namespace cattle-system --set hostname=rancher.192.168.122.64.nip.io --set replicas=1
```

2.6 이전 버전으로 설치하기를 원할 경우, 아래의 --version 옵션을 사용해서 설치 진행 가능

<aside>
ℹ️ # helm search repo rancher-stable -l# helm install rancher rancher-stable/rancher --namespace cattle-system --set [hostname=](http://hostname=rancher.l-cloud.co.kr/)[rancher.202.30.164.200.nip.io](http://rancher.202.30.164.200.nip.io/) --version=2.5.9 --set replicas=1

</aside>

<aside>
ℹ️ helm repo 모든 리스트 보기# helm search repo rancher-stable -l

</aside>

### K9S 설치

```java
wget -qO- https://github.com/derailed/k9s/releases/download/v0.24.15/k9s_Linux_x86_64.tar.gz | tar zxvf -  -C /tmp/
mv /tmp/k9s /usr/local/bin
```

