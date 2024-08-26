# Kubernetes Cluster Config

개인 서버 on premise 환경으로 클러스터 구성
`kubeadm`으로 구성

> Debian 계열 Linux 환경 기준

- raspberry pi 4 (2대)
- raspberry pi 5 (2대)

(raspberry 기본 설정은 생략)

<br>

## 📌 서버 컴퓨터 설정

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


이어서 작성할 것