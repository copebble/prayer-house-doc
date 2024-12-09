# Kubernetes Cluster Config

개인 서버 on premise 환경으로 클러스터 구성
`kubeadm`으로 구성

> Debian 계열 Linux 환경 기준

- raspberry pi 4 (2대)
- raspberry pi 5 (2대)

(raspberry 기본 설정은 생략)

> All Nodes: 클러스터에 참여하는 모든 노드들 대상(control-plane, worker 모두 포함)
> Control Plane: only control plane 대상

<br>

## References

참고 링크 먼저 첨부 (해당 글 보고 진행해도 무방)

- [k8s kubeadm 진행 방법 공식 문서](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
- [kubeadm으로 설치해보기 (blog)](https://velog.io/@moonblue/kubeadm-%EC%84%A4%EC%B9%98%ED%95%98%EA%B8%B0)
- [raspberry pi kubeadm 클러스터 설치 (blog)](https://velog.io/@2h-kim/RaspberryPI-Kubernetes-%EA%B5%AC%EC%B6%95)

<br>

## 📌 서버 컴퓨터 설정 (All Nodes)

- swap off
- cgroup 설정

### swap off

쿠베로 구성할 모든 노드들에 메모리 Swap 비활성화 해야 한다.
memory swap 활성화시 각 노드별 컨테이너 성능 일관성이 깨질 수 있어서 비활성화 처리한다고 한다. 
(쿠베 클러스터에 참여하는 모든 노드들은 일관된 성능을 가지고 있는 것이 좋다고 한다.)

```shell
sudo dphys-swapfile swapoff
sudo dphys-swapfile uninstall
sudo update-rc.d dphys-swapfile remove
sudo apt purge dphys-swapfile -y

free -m

sudo systemctl reboot
```

### cgroup 설정

raspberry pi는 기본적으로 cgroup 기능을 비활성화해두는데 해당 설정을 활성화해야 한다.
안 그러면 `kubeadm init` 수행하는데 있어 오류가 발생할 것이다.

```shell
...
CGROUPS_CPUSET: enabled
CGROUPS_DEVICES: enabled
CGROUPS_FREEZER: enabled
CGROUPS_MEMORY: missing # 오류
CGROUPS_PIDS: enabled
...
```

```shell
$ stat -fc %T /sys/fs/cgroup/
cgroup2fs
```
먼저 사용하고 있는 os에서 cgroup v1, v2인지 체크필요
v2 이면 `systemd.unified_cgroup_hierarchy=1`를 추가하면 된다.
v1 이면 `cgroup_memory=1 cgroup_enable=memory` 이것만 작성

`cmdline.txt`에 새롭게 내용 추가해야 한다.
```shell
sudo vim /boot/firmware/cmdline.txt

[...기존 내용...] cgroup_memory=1 cgroup_enable=memory (systemd.unified_cgroup_hierarchy=1)
```

> 주의 할 점은 해당 cmdline.txt 파일은 반드시 한 줄로 append해서 작성해야 한다.
> 새 줄로 해서 설정했다가 booting이 안돼서 다시 os를 설치해야 하는 불상사 발생할 수 있음

<br>

## 📌 서버 내 k8s repository 설정 (All Nodes)

k8s v1.31

> 공식문서 참고
> https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

```shell
# k8s repository 설정에 필요한 패키지 설치
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

# k8s repository용 public signing key 다운로드
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
```
위 내용은 공식 문서에 나와 있는 내용

<br>

## 📌 Install kube command program (All Nodes)

k8s cluster 구성을 위한 패키지 설치 진행해야 함

```shell
wget -qO- get.docker.com | sh
```
docker 설치 해야 함

```shell
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```
k8s cluster 구성을 위한 kubeadm 패키지와 cluster 구성 및 관리에 필수인 kubelet, kubectl 설치

<br>

## 📌 CRI 구성 (All Nodes)

[Container Runtime Interface](https://kubernetes.io/ko/docs/concepts/architecture/cri/)

> You need a working container runtime on each Node in your cluster, so that the kubelet can launch Pods and their containers.

모든 노드들에 CRI 구성이 필수다. 그래야 kubelet이 각 노드들을 컨트롤 할 수 있는 것 같다.

```shell
sudo apt install containerd
# 없는 경우에만
sudo mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml

# config.toml 파일에서 SystemdCgroup 설정값 확인 및 변경
# SystemdCgroup 값이 false 라면 true 로 변경해야 함
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml

# containerd.service 변경사항 적용
sudo systemctl restart containerd
```

<br>

## 📌 Control Plane 설정 (Control Plane)

```shell
kubeadm init \
--apiserver-advertise-address={Control Plane 동작할 서버 ip}  \
--pod-network-cidr=192.168.0.0/16
```
- `pod-network-cidr`
  - 컨테이너의 네트워크 대역
  - `192.168.0.0/16`: 해당 ip 대역으로 한 것은 조금 있다가 설치할 calico의 default 네트워크 대역이기 때문. 다른 걸로 해도 상관 없다.

```
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

sudo kubeadm join [server_url]:6443 --token [...] \
	--discovery-token-ca-cert-hash sha256:[...]
```
성공적으로 적용되면 위와 같은 메시지가 나오는데 그대로 하면 된다.

```shell
sudo kubeadm join [server_url]:6443 --token [...] \
	--discovery-token-ca-cert-hash sha256:[...]
```
그리고 각 worker node에 위 명령어 입력하면 된다.

### rejoin 하고자 할 때 (worker node)

```shell
kubeadm token list
kubeadm token create

openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
```
- [reference link](https://velog.io/@numerok/kubeadm-join%EC%9C%BC%EB%A1%9C-%ED%81%B4%EB%9F%AC%EC%8A%A4%ED%84%B0%EC%97%90-%EB%85%B8%EB%93%9C-%EC%B6%94%EA%B0%80)

### trouble shooting
```
[ERROR FileContent--proc-sys-net-ipv4-ip_forward]: /proc/sys/net/ipv4/ip_forward contents are not set to 1
```

```shell
sudo vim /etc/sysctl.conf
## 아래 주석 해제
#net.ipv4.ip_forward=1
```

```shell
ping [control-plane node ip]
Destination Port Unreachable
```
ping 날렸을 때 위와 같이 control-plane node 간 통신이 안될 수 있다.

```shell
sudo iptables -L -n -v
# KUBE-IPVS-FILTER 정보 확인
```
kube-proxy 설정에서 mode가 ipvs로 세팅되어 있어서 iptable filter 자동 생성.
해당 filter에 의해 worker-node <> control-plane 간에 통신이 안될 수 있음

```shell
# control-plane node 에서
kubectl -n kube-system edit configmap kube-proxy
# mode: "ipvs" >> "iptables" 변경
```
위와 같이 `iptables`로 configmap 변경하고 kubeadm join 실행해야 한다.

```shell
# worker node
# kubeadm 초기화 필요할 때
sudo kubeadm reset
```
reset 한 번 하고 reboot 했다가 다시 join 하는 것을 추천

<br>

## 📌 Calico 설치 (Control Plane)

각 worker node에 join까지 완료되었다면 control plane 서버에서 `kubectl get pods` 해보면 각 node가 NotReady로 나올 것이다.

추가로 플러그인을 적용해야 하는데 다음 과정 진행하면 된다.

쿠베 컨테이너 간 통신을 위한 플러그인 3개 중 하나

- 공식 문서 참고
    https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart

해당 플러그인이 설치되어 있어야 각 노드 간에 서로 통신이 가능한 상태가 된다.

```shell
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.1/manifests/tigera-operator.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.1/manifests/custom-resources.yaml
```
간단하게 설치 가능

<br> 

## 📌 Nginx Ingress Controller 설치

외부에 클러스터 내용을 오픈해야할 때 여러 객체들을 사용할텐데 그 중 `Ingress` 객체를 많이 사용하는 것 같다. 
Ingress는 routing 명세서 같은 단순 문서이고 이걸 가지고 실제 동작하는 것은 ingress controller에서 하게 된다.

Ingress를 사용할거면 Ingress Controller를 필히 설치해야 한다. 
그 중 보편적으로 쓰이는 Nginx Ingress Controller 선택

([공식 document 설치 가이드](https://kubernetes.github.io/ingress-nginx/deploy/))

```shell
kubectl apply -f kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.11.2/deploy/static/provider/baremetal/deploy.yaml
```
간단하게 apply yaml로 설정(정말 간단)
그리고 공식 문서에서 가이드해주는대로 Local test 해볼 것을 추천
**(본인은 직접 서버 구성해서 cluster를 설치한 유형이기 때문에 baremetal 링크로 설치)**

### nodeSelector

nginx ingress controller를 설치하면 `ingress-nginx-controller`라는 deployment가 생기고 여기에 묶여 있는 pod가 뜨게 된다.

만약 특정 node에 pod를 위치시키고 싶으면 nodeSelector나 affinity를 사용하면 되는데
내 프로젝트는 4대의 컴퓨팅 자원을 적절히 분배해야 하기에 부득이 하게 control plane 속한 노드에 위치시켜야 한다.

문제는 control-plane node를 보면 taint가 걸려있다.

```shell
$ kubectl describe node raspberrypi-5-uel
Taints: node-role.kubernetes.io/control-plane:NoSchedule
```
`NoSchedule` 걸려 있다. (하드하게 해당 노드에 스케줄 되지 않도록 설정)

그래서 `ingress-nginx-controller` deployment yaml 스펙 수정할 때
toleration 추가했다.

```yaml
apiVersion: v1
kind: Deployment
metadata:
  #...
  name: ingress-nginx-controller
  namespace: ingress-nginx
spec:
  #...
  nodeSelector:
    kubernetes.io/hostname: [원하는 노드 이름]
  tolerations:
  - effect: NoSchedule
    key: node-role.kubernetes.io/control-plane
    operator: Equal
  #...
```
위와 같이 `nodeSelector`에 hostname이든 뭐든 스케줄 되기 원하는 노드에 대한 내용을 명시하고
`tolerations` 통해 taint를 무시하도록 설정하면 control-plane으로 스케줄링 되게끔 할 수 있다.

### Service 객체 설정

baremetal 방식으로 ingress controller 설치하고 나면 다음과 같이 `NodePort` 유형으로 Service 객체가 생성이 되어 있을 것이다.
```shell
kubectl -n ingress-nginx get svc
kubectl -n ingress-nginx edit svc ingress-nginx-controller


```
type을 `LoadBalancer`로 바꾸면 된다.
어떻게 보면 온프레미스 방식인 본인 프로젝트 환경에서는 어울리지 않을 수 있다. (실제로 공식 문서에서는 `NodePort`로 설정되어 있음)
LoadBalancer를 통해 특정 포트에 국한되지 않고 ip를 할당받아서 연동할 수 있다는 점에서 선택하게 됨

> LoadBalancer는 사실 AWS 같은 클라우드 환경에서 사용되는 것이지 온프레미스 방식에서는 사용되지 않으나 사용하려면 MetalLB 같은 플러그인이 있어야 한다. MetalLB는 따로 설정한 ip 범위 안에서 LoadBalancer에 ip를 할당해주는 기능을 하게 된다.

### 그 외 옵션 (좀 더 파헤쳐봐야 하는 부분)

`ingress-nginx-controller` Service 객체에 신박한 옵션이 있는데 아직 완전히 이해가 가지 않음

- `externalTrafficPolicy`
  - `Cluster` (default):
    - 외부에서 들어오는 트래픽이 클러스터 내 모든 노드에 있는 파드로 분산
    - 클러스터의 모든 노드가 서비스 엔드포인트로 동작할 수 있다.
    - 클라이언트의 원래 IP 주소는 유지되지 않으며, 대신 클러스터 노드의 IP 주소로 대체
      이 때문에, 원래 클라이언트 IP를 필요로 하는 로그나 방화벽 규칙에 문제가 있을 수 있다.
  - `Local`:
    - 외부에서 들어오는 트래픽이 해당 서비스 엔드포인트를 가진 노드로만 전달
    - 클라이언트의 원래 IP 주소가 유지. 이는 원본 IP 주소 기반의 접근 제어나 로깅을 구현해야 할 때 유용
    - 단, 서비스 엔드포인트가 없는 노드는 트래픽을 받지 않으므로 트래픽 분산이 덜 효율적일 수 있음
    - 따라서 노드가 고르게 분포되어 있지 않으면 일부 노드에 부하가 집중될 수 있다.
- `internalTrafficPolicy`

<br>

## 📌 Metal LB Installation

[MetalLB doc(installation)](https://metallb.universe.tf/installation/)

```shell
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.8/config/manifests/metallb-native.yaml

kubectl -n metallb-system get all
```
k8s cluster에 metallb 관련 쿠베 객체들이 설치가 되어 있으면 된다.
여기에 간단하게 ip address pool만 설정하면 된다.

```shell
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: metallb-ip-address-pool
  namespace: metallb-system
spec:
  addresses:
  - [원하는 ip]-[원하는 ip]
```
프로젝트 인프라 환경상 특정 노드 기기 하나에서 외부 트래픽을 받을 것이기에 
**해당 기기의 실제 private ip로 설정함(baremetal 환경에 해당)**

<br> 

## 📌 Credential 등록

작성할 것

```shell
kubectl -n admin create serviceaccount beanie

kubectl create clusterrolebinding cluster-admin-beanie \
    --clusterrole=cluster-admin \
    --serviceaccount=admin:beanie
```
위의 ServiceAccount 이름, namespace는 임의로 설정 가능

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: beanie-account-token
  namespace: admin
  annotations:
    kubernetes.io/service-account.name: beanie
type: kubernetes.io/service-account-token
```
Secret 생성

<br> 

## 📌 Context 등록

```shell
kubectl config set-credentials [USER_NAME] --token=[TOKEN]
```
token은 위에서 생성한 Secret에서 확인 가능

```shell
kubectl config set-cluster [CLUSTER_NAME] --insecure-skip-tls-verify=true --server [MASTER_NODE_HTTPS_URL]
kubectl config set-context [CONTEXT_NAME] --cluster=[CLUSTER_NAME] --user=[USER_NAME]
kubectl config use-context [CONTEXT_NAME]
```
`MASTER_NODE_HTTPS_URL` : https로 해야 한다.
`--insecure-skip-tls-verify=true` : 따로 cluster에 tls 인증 설정이 안되어 있으면 true로 해야하는 듯

> 특히 홈 네트워크에 k8s cluster를 구성한 경우 
> 도메인 연동해서 외부에 노출해야 할 때 라우터에 `6443` 포트 대상으로 포트 포워딩 해야 한다.

<br>

## 📌 worker node shutdown시 대응

```shell
ping [master node ip address]
```
우선 master node의 ip address로 ping을 보내 icmp 정상적으로 통신되는지 체크

혹여나 문제가 생겨서 worker node 끊겼을 시 
```shell
kubeadm 
```