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
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
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
# 없는 경우에만
mkdir -p /etc/containerd
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
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.11.2/deploy/static/provider/cloud/deploy.yaml
```
간단하게 apply yaml로 설정(정말 간단)
그리고 공식 문서에서 가이드해주는대로 Local test 해볼 것을 추천

<br> 

## 📌 Credential 등록

작성할 것

<br> 

## 📌 Context 등록

작성할 것