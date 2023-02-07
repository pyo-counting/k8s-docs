**Note:** k8s 1.24 버전부터 Dockershim은 삭제됐다.

각 no에서 Po가 실행될 수 있도록 container runtime이 설치되어야한다.

k8s 1.24에서는 CRI를 따르는 runtime을 사용해야 한다.

이 페이지에서는 k8s에서 사용할 수 있는 몇몇 container runtime에 대해 설명한다.

- containerd
- CRI-O
- Docker Engine
- Mirantis Container Runtime

**Note:**

## Install and configure prerequisites
아래 각 단계는 lunux에서의 공통적인 k8s 설정이다.

필요하지 않은 경우 건너뛰어도 된다.

container runtime과 관련해 추가적인 내용은 [Network Plugin Requirements](https://v1-24.docs.kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/#network-plugin-requirements) 페이지를 참고한다.

### Forwarding IPv4 and letting iptables see bridged traffic
`lsmod | grep br_netfilter` 명령어를 사용해 br_netfilter 모듈이 로드됐는지 확인한다.

명시적으로 로드하기 위해 `sudo modprobe br_netfilter` 명령어를 실행한다.

In order for a Linux node's iptables to correctly view bridged traffic, verify that net.bridge.bridge-nf-call-iptables is set to 1 in your sysctl config. 아래는 예시다.

``` bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system
```

## Cgroup drivers
리눅스에서 프로세스에 할당된 리소스를 제한하기 위해 control groups를 사용한다.

리눅스 distribution에서 systemd가 init system으로 사용되면, init 프로세스는 root control group을 생성 및 사용하면서 cgroup 관리자 역할을 수행한다. systemd는 cgroup과 통합되어 있으며 systemd 당 cgroup을 할당한다. container runtime과 kubelet이 cgroupfs를 사용하도록 설정할 수 있다. systemd과 함께 cgroupfs를 사용하면 두 개의 다른 cgroup 관리자가 있다는 것을 의미한다.

cgroup driver는 cgroup을 관리하는 모듈을 의미한다. cgroup driver는 cgroupfs driver와 systemd driver가 존재한다. cgroupfs driver는 자신이 직접 cgroupfs을 통해서 cgroup을 제어한다. 반면 systemd driver는 systemd를 통해서 cgroup을 제어합니다.

단일 cgroup 관리자는 할당된 리소스 뷰가 단순하며, 사용 중인 리소스와 사용 가능한 리소스를 보다 일관되게 볼 수 있다. 시스템에 cgroup 관리자가 둘일 경우 해당 리소스에 대한 두가지 뷰가 있게된다. kubelet과 docker에 cgroupfs를 사용하도록 설정한 경우 리소스 pressure에 대한 불안정한 현상이 있다는 사람들의 보고가 있다.

container runtime, kubelet이 cgroup driver로 systemd를 사용하도록 설정을 변경하면 시스템이 안정화된다. docker에서 이를 설정하기 위해 native.cgroupdriver=systemd를 설정한다.

### Cgroup version 2

### Migrating to the systemd driver in kubeadm managed clusters

## CRI version support
container runtime은 최소 v1alpha2 CRI를 지원해야 한다.

k8s 1.24는 기본적으로 CRI API의 v1을 사용한다. container runtime이 v1 API를 지원하지 않는다면 대신 v1alpha2(deprecated) API를 사용한다.

## Container runtimes

### containerd

### CRI-O

### Docker Engine
**Note:** 아래에서는 k8s의 container rumtime으로 Docker Engine을 사용하기 위해 cri-dockerd 어댑터를 사용한다고 장헌다.

1. 각 no에서 Docker를 설치한다.
2. cri-dockerd를 설치한다.

cri-dockerd의 경우, CRI socket은 기본적으로 `/run/cri-dockerd.sock`이다.

#### Overriding the sandbox(pause) image
cri-dockerd 어댑터는 po의 infrastructure container("pause image")로 사용하기 위한 container image를 CLI --pod-infra-container-image flag를 통해 설정할 수 있다. 

### Mirantis Container Runtime