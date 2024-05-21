> **Note**:  
> k8s 1.24 버전부터 Dockershim은 삭제됐다.

각 no에서 po가 실행될 수 있도록 container runtime이 설치되어야 한다.

k8s 1.30에서는 CRI(Container Runtime Interface)를 따르는 runtime을 사용해야 한다.

이 페이지에서는 k8s에서 사용할 수 있는 몇몇 container runtime에 대해 설명한다.
- containerd
- CRI-O
- Docker Engine
- Mirantis Container Runtime

> **Note**:  
> Kubernetes releases before v1.24 included a direct integration with Docker Engine, using a component named dockershim. That special direct integration is no longer part of Kubernetes (this removal was announced as part of the v1.20 release). You can read Check whether Dockershim removal affects you to understand how this removal might affect you. To learn about migrating from using dockershim, see Migrating from dockershim.

## Install and configure prerequisites
### Network configuration
기본적으로 linux kernel은 IPv4 패킷이 network interface 간 라우팅되는 것을 허용하지 않는다. 대부분의 k8s cluster netwokring 구현은 필요한 경우 이 설정을 변경하지만 일부는 관리자가 직접 변경해야 한다(Some might also expect other sysctl parameters to be set, kernel modules to be loaded, etc; consult the documentation for your specific network implementation).

### Enable IPv4 packet forwarding
아래 명령어를 실행한다.
``` sh
# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system
```

아래 명령어를 사용해 net.ipv4.ip_forward가 1로 설정됐는지 확인한다.
``` sh
sysctl net.ipv4.ip_forward
```

## cgroup drivers
리눅스에서 프로세스에 할당된 리소스를 제한하기 위해 control groups를 사용한다.

kublet과 container runtime 모두 control group을 통해 po, container에 대한 리소스 관리를 수행하고 cpu/memory request, limit을 설정한다. control group을 사용하기 위해 kublet과 container runtime은 cgroup driver를 사용해야 한다. kublet과 container runtime이 동일한 cgroup driver를 사용하고 동일하게 구성되는 것이 중요하다.

2가지 cgroup driver가 존재한다.
- cgroupfs
- systemd

cgroup driver는 cgroup을 관리하는 모듈을 의미한다. cgroup driver는 cgroupfs driver와 systemd driver가 존재한다. cgroupfs driver는 자신이 직접 cgroupfs을 통해서 cgroup을 제어한다. 반면 systemd driver는 systemd를 통해서 cgroup을 제어합니다.

참고
- https://tech.kakao.com/2020/06/29/cgroup-driver/

### cgroupfs driver
cgroupfs는 kubelet의 기본 cgroup driver다. cgroupfs를 사용하면 kubelet, container runtime은 직접 cgroup 파일시스템에 접근해 cgroup을 구성한다.

[systemd](https://www.freedesktop.org/wiki/Software/systemd/)가 init system인 경우 cgroupfs driver를 사용하지 않는 것을 권장한다. 왜냐하면 systemd는 해당 시스템에서 자신만이 유일한 cgroup manager로 예상하기 때문이다. 추가적으로 [cgroup v2](https://kubernetes.io/docs/concepts/architecture/cgroups)를 사용하면 cgroupfs 대신 systemd를 사용해야 한다.

### systemd cgroup driver
리눅스 distribution에서 systemd가 init system으로 사용되면, init 프로세스는 root control group(cgroup)을 생성 및 사용하면서 동시에 관리자 역할을 수행한다.

systemd는 cgroup과 통합되어 있으며 systemd 당 cgroup을 할당한다. container runtime과 kubelet이 cgroupfs를 사용하도록 설정할 수 있다. systemd과 함께 cgroupfs를 사용하면 두 개의 다른 cgroup 관리자가 있다는 것을 의미한다.

단일 cgroup 관리자는 할당된 리소스 뷰가 단순하며, 사용 중인 리소스와 사용 가능한 리소스를 보다 일관되게 볼 수 있다. 시스템에 cgroup 관리자가 둘일 경우 해당 리소스에 대한 두가지 뷰가 있게된다. kubelet과 docker에 cgroupfs를 사용하도록 설정한 경우 리소스 pressure에 대한 불안정한 현상이 있다는 사람들의 보고가 있다.

이러한 불안정성을 완화하기 위해 systemd가 init systemd일 경우 container runtime, kubelet이 cgroup driver로 systemd를 사용하도록 설정을 변경하면 시스템이 안정화된다.

systemd를 group driver로 사용하기 위해 [KubeletConfiguration](https://kubernetes.io/docs/tasks/administer-cluster/kubelet-config-file/)를 설정한다.
``` yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
...
cgroupDriver: systemd
```

> **Note**:  
> Starting with v1.22 and later, when creating a cluster with kubeadm, if the user does not set the cgroupDriver field under KubeletConfiguration, kubeadm defaults it to systemd.

k8s v1.28에서 KubeletCgroupDriverFromCRI feature gate가 활성화되고 RuntimeConfig CRI RPC를 지원하는 container runtime에 대해서는 kubelet이 적절한 cgroupDriver를 자동으로 감지하고 설정 파일 내 값을 무시한다.

kubelet에 대해 systemd를 cgroup driver로 설정한 경우 container runtime에 대해서도 cgroup driver를 systemd를 사용하도록 설정해야 한다. 자세한 내용은 아래를 참고한다.
- containerd
- CRI-O

> **Caution**:  
> Changing the cgroup driver of a Node that has joined a cluster is a sensitive operation. If the kubelet has created Pods using the semantics of one cgroup driver, changing the container runtime to another cgroup driver can cause errors when trying to re-create the Pod sandbox for such existing Pods. Restarting the kubelet may not solve such errors.
> 
> If you have automation that makes it feasible, replace the node with another using the updated configuration, or reinstall it using automation.

### Migrating to the systemd driver in kubeadm managed clusters
kubeadm을 사용해 cluster를 관리하는 경우 systemd cgroup driver로 마이그레이션하는 방법은 [configuring a cgroup driver](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/configure-cgroup-driver/)을 참고한다.

## CRI version support
container runtime은 최소 v1alpha2 CRI를 지원해야 한다.

k8s 1.26는 기본적으로 CRI API의 v1을 사용한다. 이전 버전은 기본적으로 v1 버전이지만 container runtime이 v1 API를 지원하지 않는다면 kubelet은 대신 v1alpha2(deprecated) API를 사용한다.

## Container runtimes

### containerd
containerd를 CRI runtime으로 사용할 떄 고려할 내용을 설명한다.

containerd 설치는 [getting started with containerd](https://github.com/containerd/containerd/blob/main/docs/getting-started.md)을 참고한다. `config.toml` 설정 파일을 생성한 후 아래 내용을 확인한다.

리눅스 환경에서 기본 경로는 `/etc/containerd/config.toml`이다.

리눅스에서 container의 기본 CRI socket은 `/run/containerd/containerd.sock`이다.

#### Configuring the systemd cgroup driver
systemd cgroup driver를 사용하기 위해 `/etc/containerd/config.toml` 파일을 아래와 같이 설정한다.
``` toml
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  ...
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true
```

cgroup v2를 사용하는 경우 systemd cgroup driver를 권장한다.

> **Note**:  
> If you installed containerd from a package (for example, RPM or .deb), you may find that the CRI integration plugin is disabled by default.
>
> You need CRI support enabled to use containerd with Kubernetes. Make sure that cri is not included in thedisabled_plugins list within /etc/containerd/config.toml; if you made changes to that file, also restart containerd.
>
> If you experience container crash loops after the initial cluster installation or after installing a CNI, the containerd configuration provided with the package might contain incompatible configuration parameters. Consider resetting the containerd configuration with containerd config default > /etc/containerd/config.toml as specified in getting-started.md and then set the configuration parameters specified above accordingly.

설정 파일 반영을 위해 containerd를 재실행한다.
``` sh
sudo systemctl restart containerd
```

kubeadm을 사용하는 경우 [cgroup driver for kubelet](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/configure-cgroup-driver/#configuring-the-kubelet-cgroup-driver)을 참고한다.

k8s v1.28부터 alpha feature을 사용해 cgroup driver에 대한 automatic detection을 활성화할 수 있다.

#### Overriding the sandbox (pause) image 
[containerd config](https://github.com/containerd/containerd/blob/main/docs/cri/config.md)에서 아래와 같이 sandbox image을 덮어쓸 수 있다.
``` toml
[plugins."io.containerd.grpc.v1.cri"]
  sandbox_image = "registry.k8s.io/pause:3.2"
```

You might need to restart containerd as well once you've updated the config file: systemctl restart containerd.

Please note, that it is a best practice for kubelet to declare the matching pod-infra-container-image. If not configured, kubelet may attempt to garbage collect the pause image. There is ongoing work in containerd to pin the pause image and not require this setting on kubelet any longer.

### CRI-O

### Docker Engine
> **Note**:  
> 아래에서는 k8s의 container rumtime으로 Docker Engine을 사용하기 위해 [cri-dockerd](https://mirantis.github.io/cri-dockerd/) 어댑터를 사용한다고 가정헌다.

1. 각 no에서 Docker를 설치한다.
2. cri-dockerd를 설치한다.

cri-dockerd의 경우, CRI socket은 기본적으로 `/run/cri-dockerd.sock`이다.

### Mirantis Container Runtime