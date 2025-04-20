Kubernetes nodes can be scheduled to `Capacity`. 기본적으로 po는 no의 사용 가능한 모든 capacity를 소비할 수 있다. 이는 no가 일반적으로 OS와 k8s 자체를 구동하는 상당수의 system daemon을 실행하기 때문에 문제가 될 수 있다. 이러한 system daemon을 위해 자원을 따로 확보해두지 않으면 po와 system daemon이 자원을 경쟁하게 되어 no에서 리소스 고갈(resource starvation) 문제를 야기한다.

kubelet은 system daemon을 위한 compute resource를 예약하는 데 도움이 되는 'Node Allocatable'이라는 기능을 제공한다. k8s는 cluster 운영자가 각 no의 workload 밀도에 기반해 'Node Allocatable'을 설정하는 것을 권장한다.

## Node Allocatable
![](https://kubernetes.io/images/docs/node-capacity.svg)

no의 'Allocatable'은 po가 사용할 수 있는 compute resource의 양으로 정의된다. scheduler는 'Allocatable'을 over-subscribe하지 않는다. 현재까지 'CPU', 'memory', 'ephemeral-storage'가 지원된다.

Node Allocatable은 API의 `v1.Node` object의 일부로, 그리고 CLI에서 `kubectl describe node` 명령어로 노출된다.

kubelet은 두 가지 종류의 system daemon을 위한 resource를 예약할 수 있다.

### Enabling QoS and Pod level cgroups
no에서 node allocatable 제약을 올바르게 적용하기 위해, `.cgroupsPerQOS` 설정을 통해 cgroup hierarchy을 활성화해야 한다. 이 설정은 기본적으로 활성화되어 있다. 활성화되면, kubelet은 모든 사용자 po를 kubelet이 관리하는 cgroup hierarchy 아래에 둔다.

`.cgroupsPerQOS` 설정은 po의 QoS class 별로 cgroup hierarchy를 구성하도록 설정한다. 이 설정의 주된 목적은 po에 할당된 QoS class에 따라 자원 사용을 보다 정확하고 효과적으로 제어하고 격리하는 것이다. k8s는 po에 대해 `Guaranteed`, `Burstable`, `BestEffort` 세 가지 QoS class를 부여한다. 각 QoS class는 po의 resource request, limit 설정에 따라 결정되며 자원 부족 상황에서 po의 eviction 우선순위 등에 영향을 미친다.

`.cgroupsPerQOS` 설정이 활성화되면, kubelet은 no의 system cgroup hierarchy 아래에 QoS class별로 별도의 cgroup 디렉토리를 생성한다. 예를 들어 `/sys/fs/cgroup/<subsystem>/kubepods/<qos-class>`와 같은 구조가 될 수 있다. 그리고 각 po는 해당 QoS class cgroup 하위에 자신만의 cgroup을 가지게 된다.

### Configuring a cgroup driver
kubelet은 cgroup driver를 사용해 호스트의 cgroup hierarchy 구조에 대한 조작을 지원한다. kubelet의 `.cgroupDriver`을 통해 설정 가능하다.
- `cgroupfs`는 cgroup sandbox 관리를 위해 호스트의 cgroup 파일 시스템을 직접 조작하는 기본 driver이다.
- `systemd`는 해당 init system이 지원하는 리소스에 대해 transient slice를 사용하여 cgroup sandbox를 관리하는 대안 driver이다.

연관된 container runtime의 설정에 따라 관리자는 올바른 시스템 동작을 보장하기 위해 동일한 cgroup driver를 선택해야 한다. 예를 들어 관리자가 containerd runtime에서 제공하는 systemd cgroup driver를 사용한다면 kubelet도 systemd cgroup driver를 사용하도록 설정해야 한다.

### Kube Reserved
`.kubeReserved`는 kubelet, container runtime 등과 같은 k8s 관련 system daemon을 위한 resource 예약을 포착하기 위한 것이다. po로 실행되는 system daemon이 대상이 아님을 유의한다. `.kubeReserved`는 일반적으로 po 밀도에 따라 달라진다(po가 많을 수록 k8s 관련 system daemon이 관리해야 할 부하가 커지므로).

cpu, memory, ephemeral-storage 외에도, k8s system daemon을 위해 지정된 수의 process ID를 예약하기 위해 pid를 지정할 수 있다.

k8s system daemon에 `.kubeReserved`를 강제하기 위해 `.kubeReservedCgroup` 설정에 kube daemon의 부모 control group을 지정하고 `.enforceNodeAllocatable`에 kube-reserved 값을 추가한다.

k8s system daemon은 최상위 control group 아래에 배치하는 것이 권장된다 (예: systemd 환경에서는 runtime.slice). 각 system daemon은 이상적으로는 자체적인 자식 control group 내에서 실행되어야 한다. 권장되는 control group 계층 구조에 대한 자세한 내용은 [the design proposal](https://github.com/kubernetes/design-proposals-archive/blob/main/node/node-allocatable.md#recommended-cgroups-setup)을 참고한다.

kubelet은 `.kubeReservedCgroup`가 존재하지 않으면 생성하지 않는다는 점에 유의해야 한다. 유효하지 않은 cgroup이 지정되면 kubelet은 시작에 실패한다. 따라서 cluster 관리자는 필요하다면 kubelet을 시작하기 전에 해당 cgroup 경로를 호스트 시스템에 미리 생성해야한다. systemd cgroup driver은 cgroup 이름에 패 턴이 있다(이름에 `.slice` 접두사가 붙는다).

### System Reserved
`.systemReserved`는 sshd, udev 등과 같은 OS system daemon을 위한 resource 예약을 포착하기 위한 것이다. 현재 k8s은 po에 kernel memory가 계산되지 않으므로 `.systemReserved`는 kernel을 위한 memory도 예약해야 한다. 사용자 로그인 세션을 위한 resource를 예약하는 것도 권장된다 (systemd 환경에서는 user.slice).

cpu, memory, ephemeral-storage 외에도, OS system daemon을 위해 지정된 수의 process ID를 예약하기 위해 pid를 지정할 수 있다.

system daemon에 `.systemReserved`를 선택적으로 강제하려면, `.systemReservedCgroup` 설정 값으로 OS system daemon의 부모 control group을 지정하고, `.enforceNodeAllocatable`에 system-reserved 값을 추가한다.

OS system daemon은 최상위 control group 아래에 배치하는 것이 권장된다 (예: systemd 머신에서는 system.slice).


kubelet은 `.systemReservedCgroup`가 존재하지 않으면 생성하지 않는다는 점에 유의해야 한다. 유효하지 않은 cgroup이 지정되면 kubelet은 실패한다. 따라서 cluster 관리자는 필요하다면 kubelet을 시작하기 전에 해당 cgroup 경로를 호스트 시스템에 미리 생성해야한다. systemd cgroup driver은 cgroup 이름에 패 턴이 있다(이름에 `.slice` 접두사가 붙는다).

### Explicitly Reserved CPU List
`.reservedSystemCPUs`는 OS system daemon과 k8s system daemon을 위한 명시적인 CPU set을 정의하기 위한 것이다. `.reservedSystemCPUs`는 cpuset resource와 관련하여 OS system daemon과 k8s system daemon을 위해 별도의 최상위 cgroup을 정의할 의도가 없는 시스템을 위한 것이다. 만약 kubelet 설정에 `.kubeReservedCgroup`와 `.systemReservedCgroup`가 설정되어 있지 않다면, `.reservedSystemCPUs`에 지정된 CPU set이 system daemon이 사용할 CPU를 결정하는 데 있어 우선권을 가진다. 하지만 `.kubeReservedCgroup`와 `.systemReservedCgroup`가 설정되고 해당 cgroup에 cpuset이 정의되어 있다면, cgroup 설정이 우선 순위가 높을 수도 있다.

이 옵션은 제어되지 않는 interrupts/timers가 workload 성능에 영향을 미칠 수 있는 Telco/NFV 사용 사례를 위해 특별히 설계되었다. 이 옵션을 사용하여 system/k8s daemon뿐만 아니라 interrupts/timers를 위한 명시적인 cpuset을 정의할 수 있으며, 이를 통해 시스템의 나머지 CPU들을 workload 전용으로 사용하여 제어되지 않는 interrupts/timers의 영향을 덜 받을 수 있다. `.reservedSystemCPUs` 설정은 어떤 CPU를 예약할지를 kubelet에게 알려주는 역할만 한다. 실제로 OS system daemon, k8s daemon, 그리고 interrupts/timers를 해당 예약된 CPU set으로 이동시키고 고정하는 작업은 kubelet 스스로 수행하지 않는다. system daemon, k8s daemon 및 interrupts/timers를 이 옵션에 의해 정의된 명시적인 cpuset으로 이동시키려면 k8s 외부의 다른 메커니즘을 사용해야 한다. 예를 들어: Centos에서는 tuned 도구 세트를 사용하여 이를 수행할 수 있다.

### Eviction Thresholds
no 수준의 memory pressure는 system OOM(Out-Of-Memory)으로 이어지며 이는 no와 그 위에서 실행되는 모든 po에 영향을 미친다(이는 운영체제 kernel 수준의 문제로 no가 응답 불능 상태가 되거나 재부팅되는 등 심각한 장애로 이어질 수 있다). no는 memory가 회수될 때까지 일시적으로 오프라인 상태가 될 수 있다. system OOM을 피하기 (또는 발생 확률을 줄이기) 위해 kubelet은 out of resource(node-pressure eviction) 관리 기능을 제공한다. eviction은 memory와 ephemeral-storage에 대해서만 지원된다. `.evictionHard` 설정을 통해 일부 memory를 예약함으로써, node의 memory 가용성이 예약된 값 아래로 떨어질 때마다 kubelet은 po를 eviction시키려고 시도한다. 만약 no에 system daemon이 존재하지 않는다면 pod는 capacity - eviction-hard 값까지 리소스를 사용할 수 있다. 즉, eviction을 위해 예약된 리소스는 po가 사용할 수 없게된다.

### Enforcing Node Allocatable
scheduler는 'Allocatable'을 po가 사용할 수 있는 가용 capacity로 취급한다.

kubelet은 기본적으로 모든 po에 대해 'Allocatable'을 강제한다(enforce). 강제는 모든 po의 전체 사용량이 'Allocatable'을 초과할 때마다 po를 eviction 시키는 방식으로 수행된다. eviction 정책에 대한 자세한 내용은 [node pressure eviction](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/)을 참고한다. 이 강제 적용은 kubelet의 `.enforceNodeAllocatable` 설정에 pods 값을 지정하면 된다.

선택적으로, 동일한 설정에 kube-reserved, system-reserved 값을 지정하여 kubelet이 `.kubeReserved`, `.systemReserved`를 강제하도록 할 수 있다. `.kubeReserved`, `.systemReserved`를 강제하려면 각각 `.kubeReservedCgroup`, `.systemReservedCgroup`가 지정되어야 한다는 점에 유의해야 한다.

## General Guidelines
system daemon(OS system daemon과 k8s system daemon )은 Guaranteed Pod와 유사하게 취급되어야 한다. system daemon은 경계가 설정된 control group 내에서 burst(일시적으로 자원을 더 많이 사용하는 것)할 수 있으며, 이러한 동작은 k8s 배포의 일부로 관리되어야 한다. 예를 들어, kubelet과 container runtime은 함께 `.kubeReserved`에 정의된 자원을 공유할 수 있다. 하지만 `.enforceNodeAllocatable`에 kube-reserved를 포함시켜 `.kubeReserved`가 강제되면, kubelet을 포함한 k8s system daemon들은 `.kubeReserved`에 설정된 총 자원량 이상으로 burst하여 no의 사용 가능 자원을 모두 소비할 수는 없다. 이는 예약된 자원 범위를 벗어나 다른 po나 OS system daemon의 자원을 침범하는 것을 막는다.

`.systemReserved`에 정의된 OS system daemon을 위한 자원 예약을 강제로 적용하는 것은 매우 신중하게 접근해야 한다. 이는 OS 운영에 필수적인 서비스(예: sshd, systemd 자체 등)의 자원이 제한될 수 있기 때문이다. 만약 `.systemReserved` 설정이 너무 타이트하면, 해당 system daemon이 CPU를 충분히 확보하지 못하거나(CPU starvation), memory 부족으로 OOM kill 되거나, 심지어 새로운 프로세스를 생성하지 못하는(unable to fork) 상황이 발생할 수 있다. 이러한 문제는 node 전체의 장애로 이어질 수 있는 매우 심각한 상황이다. 따라서 `.systemReserved`를 강제하려면, 사전에 해당 no의 자원 사용 패턴을 철저히 분석하여 정확한 필요량을 예상하고, 만약 system daemon이 문제가 발생하더라도 신속하게 복구할 수 있는 조건이 됐을 때만 수행하는 것을 권장된다. 이와 관련해 아래 단계적 적용을 권장한다.

1. 우선 `.enforceNodeAllocatable`을 po에 대해 강제하는 것으로 시작한다. 이는 po가 예약된 영역에서 벗어나지 못하도록하는 기본적인 보호 장치다.
2. kube system daemon을 추적하기 위한 적절한 모니터링 시스템이 갖춰지면, 사용량 heuristics에 기반하여 `.kubeReserved`에 대한 강제도 시도한다.
3. 정말 필요한 경우에만 시간이 지남에 따라 `.systemReserved`를 강제한다.

kube system daemon의 resource 요구 사항은 더 많은 기능이 추가됨에 따라 시간이 지남에 따라 증가할 수 있다. 시간이 지나면서 k8s 프로젝트는 system daemon의 사용률을 낮추려고 시도하겠지만, 현재로서는 우선순위가 아니다. 따라서 향후 릴리스에서는 'Allocatable' capacity의 감소를 예상해야 한다.

## Example Scenario
아래는 no의 Allocatable 계산 예시다.
-  node는 32Gi의 memory, 16 CPUs 및 100Gi의 Storage를 가지고 있다.
-  `.kubeReserved`는 {cpu: 1000m, memory: 2Gi, ephemeral-storage: 1Gi}로 설정되어 있다.
- `.systemReserved`는 {cpu: 500m, memory: 1Gi, ephemeral-storage: 1Gi}로 설정되어 있다.
- `.evictionHard`는 {memory.available: "<500Mi", nodefs.available: "<10%"}로 설정되어 있다.

이 시나리오에서 'Allocatable'은 14.5 CPUs, 28.5Gi의 memory, 88Gi의 로컬 storage가 될 것이다. scheduler는 이 node의 모든 po의 전체 memory requests 합이 28.5Gi를 초과하지 않고 storage가 88Gi를 초과하지 않도록 보장한다. 그리고 kubelet은 po의 전체 memory 사용량이 28.5Gi를 초과하거나, 전체 디스크 사용량이 88Gi를 초과할 때마다 pod를 eviction한다. 그리고 pod들은 합쳐서 14.5 CPUs보다 더 많이 사용할 수 없다.

만약 `.kubeReserved`, `.systemReserved`가 강제되지 않아(`.enforceNodeAllocatable` 설정에서 제외) system daemon이 자신들의 예약을 초과 사용한다면 kubelet은 전체 no memory 사용량이 31.5Gi보다 높거나 storage가 90Gi보다 클 때마다 po를 eviction한다.