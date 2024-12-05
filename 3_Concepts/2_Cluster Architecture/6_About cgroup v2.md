linux에서는 control group을 사용해 프로세스에 할당된 리소스를 관리한다.

kubelet과 contaienr runtime은 cgroup interface를 사용해야 하며 이를 통해 [resource management for pods and containers ](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)를 수행한다.

linux에는 두 가지 버전의 cgroup이 있다. cgroup v1, cgroupv2. cgroup v2는 cgroup API의 차세대 버전이다.

## What is cgroup v2?
cgroup v2는 linux cgroup API 차세대 버전이다. cgroup v2는 향상된 리소스 관리 기능을 갖춘 통합된 제어 시스템을 제공한다.

cgroup v2는 다음과 같은 여러 가지 개선 사항을 제공한다.
- single unified hierarchy API
- container에 대한 안전한 sub-tree delegation
- [Pressure Stall Information](https://www.kernel.org/doc/html/latest/accounting/psi.html)과 같은 새로운 기능
- 다양한 리소스 간 향상된 리소스 할당 및 격리
    - 다른 유형의 메모리 할당에 대한 통합된 accounting (네트워크 메모리, 커널 메모리 등)
    - page cache write back 등과 같은 즉시 발생하지 않는 리소스 변경에 대한 accounting

k8s의 일부 기능은 향상된 리소스 관리, 격리를 위해 cgroup v2를 전용으로 사용한다. 예를 들어, [MemoryQoS](https://kubernetes.io/blog/2021/11/26/qos-memory-resources/) 기능은 메모리 QoS를 향상시키고 cgroup v2 기본 요소에 의존한다.

## Using cgroup v2
cgroup v2를 사용에 대한 권장 방법은 기본적으로 cgroup v2가 활성화 및 사용하는 linux 배포판을 사용하는 것이다.

linux 배포판이 cgroup v2를 사용하는지 확인하기위해 아래를 참고한다.

### Requirements
cgroup v2는 아래 요구사항이 있다.
- OS distribution enables cgroup v2
- Linux Kernel version is 5.8 or later
- Container runtime supports cgroup v2. For example:
    - containerd v1.4 and later
    - cri-o v1.20 and later
- The kubelet and the container runtime are configured to use the [systemd cgroup driver](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#systemd-cgroup-driver)

### Linux Distribution cgroup v2 support
cgroup v2를 사용하는 linux distribution은 [cgroup v2 documentation](https://github.com/opencontainers/runc/blob/main/docs/cgroup-v2.md)을 참고한다.
- Container Optimized OS (since M97)
- Ubuntu (since 21.10, 22.04+ recommended)
- Debian GNU/Linux (since Debian 11 bullseye)
- Fedora (since 31)
- Arch Linux (since April 2021)
- RHEL and RHEL-like distributions (since 9)

linux 배포판이 cgroup v2를 사용하는지 확인하기위해 아래를 참고한다.

linux 배포판에서 kernel cmdline boot argument를 수정해 수동으로 cgroup v2를 활성화할 수도 있다. 배포판이 GRUB를 사용하는 경우, `/etc/default/grub` 아래의 `GRUB_CMDLINE_LINUX`에 `systemd.unified_cgroup_hierarchy=1`을 추가하고 `sudo update-grub` 명령어을 실행해야 한다. 하지만 권장하는 방법은 기본적으로 cgroup v2를 활성화 및 사용하는 linux 배포판을 사용하는 것이다.

### Migrating to cgroup v2
cgroup v2로 마이그레이션하기 위해 위에서 설명한 요구 사항을 충족해야한다. 그리고 기본적으로 cgroup v2를 활성화하는 kernel 버전으로 업그레이드한다.

kubelet은 OS가 cgroup v2을 실행 중인 것을 자동으로 감지하고 추가 설정 없이 이에 맞게 동작한다.

cgroup v2로 전환했을 때 사용자는 차이점을 느끼면 안된다.

cgroup v2는 cgroup v1과 다른 API를 사용하므로 cgroup 파일 시스템에 직접 접근하는 애플리케이션이 있는 경우 cgroup v2를 지원하는 새로운 버전으로 업데이트해야 한다. 예를 들어
- 일부 third-party 모니터링, 보안 에이전트가 cgroup 파일 시스템에 의존할 수도 있다. 이러한 에이전트를 cgroup v2를 지원하는 버전으로 업데이트한다.
- po, container 모니터링을 위해 독립적인 ds로 cAdvisor를 실행하는 경우 v0.43.0 버전 이상으로 업데이트한다.
- Java 애플리케이션을 배포하는 경우, cgroup v2를 완전히 지원하는 버전을 사용하는 것이 좋다.
  - OpenJDK / HotSpot: jdk8u372, 11.0.16, 15 이상
  - IBM Semeru Runtimes: 8.0.382.0, 11.0.20.0, 17.0.8.0 이상
  - IBM Java: 8.0.8.6 이상
- [uber-go/automaxprocs](https://github.com/uber-go/automaxprocs) 패키지를 사용하는 경우 사용하는 버전이 v1.5.1 이상인지 확인한다.

## Identify the cgroup version on Linux Nodes
linux 배포판에서 cgroup 버전은 사용되는 linux 배포판, OS에 설정된 기본 cgroup 버전에 따라 다르다. 배포판이 사용하는 cgroup 버전을 확인하기 위해 아래 명령어를 수행한다.
``` sh
stat -fc %T /sys/fs/cgroup/
```

cgroup v2의 경우 `cgroup2fs`, cgroup v1의 경우 `tmpfs`이 출력된다.