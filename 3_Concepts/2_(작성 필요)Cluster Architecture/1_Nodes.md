## Management

API server에 no를 추가하기 위한 두 가지 방법이 있다:

1. no의 kubelet이 control plane에 스스로 등록
2. 사람이 직접 no object를 추가

no가 등록된 후 control plane은 새로운 no object가 유효한지 확인한다. 예를 들어 아래 JSON manifest를 통해 no를 생성할 경우:

``` json
{
  "kind": "Node",
  "apiVersion": "v1",
  "metadata": {
    "name": "10.240.79.157",
    "labels": {
      "name": "my-first-k8s-node"
    }
  }
}
```

k8s는 내부적으로 no object를 생성한다. Kubernetes checks that a kubelet has registered to the API server that matches the metadata.name field of the Node. 만약 no가 정상이면(즉, 필요한 모든 서비스가 실행 중) po를 실행할 자격이 있다. 정상이 아니라면 정상이 되기 전까지 해당 no는 클러스터와 관련된 행동에서 제외된다.

**Note**: k8s는 유효하지 않은 no의 object를 보존하면서 정상이 될때까지 게속 체크한다. health check를 멈추기 위해 no object를 직접 또는 controller가 삭제해야 한다.

no object의 이름은 DNS subdomain name 규칙을 따라야한다.

### Node name uniqeness
두 개의 no가 동시에 동일한 이름을 가질 수 없다. k8s는 동일한 이름의 resource에 대해 동일한 object로 생각한다. 동일한 이름을 갖는 no의 경우 동일한 state(네트워크, root disk 내용), label 등의 정보를 갖는다고 생각한다. 이로 인해 이름을 변경하지 않고 인스턴스가 수정되면 불일치가 발생할 수 있다. no를 교체하거나 업데이트해야 하는 경우 기존 no object를 먼저 API에서 제거하고 업데이투 후 다시 추가해야 한다.

### Self-registration of Nodes
kubelet에 --register-node flag를 true(기본 값)으로 설정하면 kubelet은 API server에 스스로 등록한다. 대부분의 경우 선호하는 방식이다.

스스로 등록할 경우 kubelet에 아래 옵션을 설정해야 한다:

- --kubeconfig: API server에 인증하기 위해 사용되는 credentials의 경로
- --cloud-provider: metadata를 읽기 위해 cloud provier와 통신하는 방법
- --register-node: API server에 스스로 등록할지 여부
- --register-with-taints: no의 taints 목록(\<key\>=\<value\>:\<effect\>를 ,로 구분)
- --node-ip: no의 IP 주소
- --node-labels: no의 label (NodeRestriction admission plugin에 의해 강제되는 label 규칙도 있다)
- --node-status-update-frequency: kubelet이 no의 상태를 API server에 보고하는 빈도

 Node authorization mode, NodeRestriction admission plugin가 활성화된 경우, kubelet은 자체 no의 resource만 생성/수정할 수 있는 권한이 있다. 

**Note**: no의 설정 변경이 필요한 경우 API server에 no를 다시 등록하는 것이 권장 방법이다. 예를 들어 kubelet이 새로운 --node-labels 옵션을 사용해 재시작 되지만 동일한 no의 이름이 사용되는 경우 label이 no 등록에 설정되어있으므로 변경 사항이 적용되지 않는다.

kubelet 재시작 시 no의 설정이 변경되는 경우 해당 no에 스케줄링된 po는 오작동하거나 문제를 일으킬 수 있다. For example, already running Pod may be tainted against the new labels assigned to the Node, while other Pods, that are incompatible with that Pod will be scheduled based on this new label. Node re-registration ensures all Pods will be drained and properly re-scheduled.

### Manual Node administration
kubelet을 사용해 no object를 생성, 수정할 수 있다.

직접 no object를 생성하기 위해 --register-node=false flag를 사용한다.

--register-node flag와 상관없이 no obejct를 수정할 수 있다. 예를 들어 label 수정할 수 있다.

no의 label을 po의 label selector와 같이 사용해 스케줄링을 제어할 수 있다.

no를 스케줄링 불가능하도록 만들면 scheduler는 해당 no에 새로운 po를 스케줄링할 수 없지만 기존 po에는 영향을 미치지 않는다. 이는 no의 재부팅, 기타 유지 보수 준비 단계를 위해 유용하다.

no를 스케줄링 불가하게 하기 위해 아래 명령어를 사용할 수 있다:

``` bash
kubectl cordon $NODENAME
```

자세한 내용은 [Safely Drain a Node](https://v1-24.docs.kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/) 페이지를 참고한다.

**Note**: ds의 일부인 po는 스케줄링 불가한 no에서도 실행될 수 있다. ds는 일반적으로 workload 애플리케이션이 제거되더라도 no에서 실행되어야 하는 no lacal 서비스다.

## Node status
no의 status는 아래 정보를 포함한다:

- Addresses
- Conditions
- Capacity and Allocatable
- Info

no의 status 및 기타 정보를 조회하기 위해 kubectl 명령어를 사용할 수 있다:

``` bash
kubectl describe node <insert-node-name-here>
```

출력의 각 상세 정보는 아래에서 설명한다. 아래는 kubectl get no -o yaml 명령어 출력 결과 중 status 부분이다.

아래는 minikube에서의 no .status 정보다:
``` yaml
  status:
    addresses:
    - address: 192.168.49.2
      type: InternalIP
    - address: minikube
      type: Hostname
    allocatable:
      cpu: "12"
      ephemeral-storage: 263174212Ki
      hugepages-2Mi: "0"
      memory: 13034336Ki
      pods: "110"
    capacity:
      cpu: "12"
      ephemeral-storage: 263174212Ki
      hugepages-2Mi: "0"
      memory: 13034336Ki
      pods: "110"
    conditions:
    - lastHeartbeatTime: "2023-02-15T13:51:10Z"
      lastTransitionTime: "2022-08-12T13:20:29Z"
      message: kubelet has sufficient memory available
      reason: KubeletHasSufficientMemory
      status: "False"
      type: MemoryPressure
    - lastHeartbeatTime: "2023-02-15T13:51:10Z"
      lastTransitionTime: "2022-08-12T13:20:29Z"
      message: kubelet has no disk pressure
      reason: KubeletHasNoDiskPressure
      status: "False"
      type: DiskPressure
    - lastHeartbeatTime: "2023-02-15T13:51:10Z"
      lastTransitionTime: "2022-08-12T13:20:29Z"
      message: kubelet has sufficient PID available
      reason: KubeletHasSufficientPID
      status: "False"
      type: PIDPressure
    - lastHeartbeatTime: "2023-02-15T13:51:10Z"
      lastTransitionTime: "2022-08-12T13:20:48Z"
      message: kubelet is posting ready status
      reason: KubeletReady
      status: "True"
      type: Ready
```

아래는 EKS에서의 no .status 정보다:
``` yaml
status:
  addresses:
  - address: 172.31.100.75
    type: InternalIP
  - address: ip-172-31-100-75.ap-northeast-2.compute.internal
    type: Hostname
  - address: ip-172-31-100-75.ap-northeast-2.compute.internal
    type: InternalDNS
  allocatable:
    attachable-volumes-aws-ebs: "39"
    cpu: 1930m
    ephemeral-storage: "47233297124"
    hugepages-1Gi: "0"
    hugepages-2Mi: "0"
    memory: 7313084Ki
    pods: "17"
  capacity:
    attachable-volumes-aws-ebs: "39"
    cpu: "2"
    ephemeral-storage: 52416492Ki
    hugepages-1Gi: "0"
    hugepages-2Mi: "0"
    memory: 8003260Ki
    pods: "17"
  conditions:
  - lastHeartbeatTime: "2023-03-27T01:40:54Z"
    lastTransitionTime: "2023-03-19T08:52:38Z"
    message: kubelet has sufficient memory available
    reason: KubeletHasSufficientMemory
    status: "False"
    type: MemoryPressure
  - lastHeartbeatTime: "2023-03-27T01:40:54Z"
    lastTransitionTime: "2023-03-19T08:52:38Z"
    message: kubelet has no disk pressure
    reason: KubeletHasNoDiskPressure
    status: "False"
    type: DiskPressure
  - lastHeartbeatTime: "2023-03-27T01:40:54Z"
    lastTransitionTime: "2023-03-19T08:52:38Z"
    message: kubelet has sufficient PID available
    reason: KubeletHasSufficientPID
    status: "False"
    type: PIDPressure
  - lastHeartbeatTime: "2023-03-27T01:40:54Z"
    lastTransitionTime: "2023-03-19T08:53:00Z"
    message: kubelet is posting ready status
    reason: KubeletReady
    status: "True"
    type: Ready
  daemonEndpoints:
    kubeletEndpoint:
      Port: 10250
  images:
  - names:
    - 602401143452.dkr.ecr-fips.us-east-1.amazonaws.com/eks/pause:3.5
    - 602401143452.dkr.ecr-fips.us-east-2.amazonaws.com/eks/pause:3.5
    - 602401143452.dkr.ecr-fips.us-west-1.amazonaws.com/eks/pause:3.5
    - 602401143452.dkr.ecr-fips.us-west-2.amazonaws.com/eks/pause:3.5
    - 602401143452.dkr.ecr.af-south-1.amazonaws.com/eks/pause:3.5
    sizeBytes: 298689
  nodeInfo:
    architecture: amd64
    bootID: 8d49e092-92ce-4b55-937e-63fd600ef01e
    containerRuntimeVersion: containerd://1.6.6
    kernelVersion: 5.4.228-131.415.amzn2.x86_64
    kubeProxyVersion: v1.24.9-eks-49d8fe8
    kubeletVersion: v1.24.9-eks-49d8fe8
    machineID: ec2f0632a9ca796563d0b6933fe8afbc
    operatingSystem: linux
    osImage: Amazon Linux 2
    systemUUID: ec2f0632-a9ca-7965-63d0-b6933fe8afbc
```

### Addresses
이 필드는 cloud provider 또는 bare metal 설정에 따라 다르다.

- HostName: no의 kernel에서 제공하는 hostname. kubelet의 --hostname-override 파라미터를 사용해 덮어쓸 수 있다.
- ExternalIP: 클러스터 외부에서 라우팅 가능한 no의 IP 주소
- InternalIP: 클러스터 내부에서 라우팅할 수 있는 no의 IP 주소

### Conditions
conditions 필드는 running no의 상태를 설명한다. conditions type의 종류는 다음과 같다:
|Node Condition    |Description|
|------------------|-----------|
|Ready             |no가 po를 실행할 수 있는 상태인 경우 True, po를 실행할 수 없는 unhealthy 상태인 경우 False, node controller가 node-monitor-grace-period(기본 값 40s) 설정 값 동안 no의 상태를 알 수 없는 경우 Unknown|
|DiskPressure      |disk 크기에 대한 pressure가 있는 경우(disk 용량이 부족할 경우), 반대의 경우 False|
|MemoryPressure    |no 메모리에 대한 pressure가 있는 경우(no의 메모리가 부족할 경우), 반대의 경우 False|
|PIDPressure       |프로세스에 대한 pressure이 있는 경우(no에 너무 많은 프로세스가 있는 경우) True, 반대의 경우 False|
|NetworkUnavailable|no의 네트워크가 옳바르게 설정되지 않은 경우 True, 반대의 경우 False|

**Note**: kubelet 명령어를 사용해 cordoned no의 상세 정보를 조회하는 경우, Condition 필드에 SchedulingDisable를 포함한다. SchedulingDisable은 k8s API 서버내 Condition이 아니다. 대신 cordoned no는 spec에 Unschedulable로 표시된다.

k8s API에서 no의 condition은 no 리소스의 .status에 표시된다. 예를 들어 아래 JSON 구조는 정상적인 no를 나타낸다:

``` json
"conditions": [
  {
    "type": "Ready",
    "status": "True",
    "reason": "KubeletReady",
    "message": "kubelet is posting ready status",
    "lastHeartbeatTime": "2019-06-05T18:38:35Z",
    "lastTransitionTime": "2019-06-05T11:41:27Z"
  }
]
```
### Capacity and Allocatable
no의 이용 가능한 리소스 정보를 제공한다: CPU, memory, no에 스케줄링 가능한 최대 po 개수

capacity 블락 내 필드는 no가 가진 총 resource의 크기를 나타낸다. available block은 일반 po에서 사용할 수 있는 no의 리소스 크기를 나타낸다.

You may read more about capacity and allocatable resources while learning how to reserve compute resources on a Node.

### Info
.status.nodeInfo 필드는 커널 버전, k8s 버전(kubelet, kube-proxy 버전), container runtime 상세 사항, no가 사용하는 OS와 같은 일반적인 정보를 제공한다. kubelet은 no로부터 이러한 정보를 수집하며 k8s API로 제공한다.

## Heartbeats
k8s no가 전송하는 heartbeat를 통해 클러스터가 사용 가능한 no를 식별할 수 있도록 도와주고 failure가 감지되면 이에 대한 조치를 취한다.

no는 2가지 heartbeat 방식을 사용한다:

- no의 .status 필드를 업데이트한다.
- kube-node-lease ns의 lease object. 각 no에 대해 lease object를 갖는다.

no의 .status를 업데이트하는 것에 비해 lease resource는 더 간단하다. heartbeat를 위해 lease를 사용하는 것은 큰 클러스터의 경우 성능에 영향을 줄여준다.

kubelet은 no의 .status를 생성 및 업데이트, lease object를 업데이트해야 하는 책임이 있다.

- kubelet은 상태가 변경되거나 설정 간격 동안 업데이트가 없는 경우 no의 .status를 업데이트 한다. .status 업데이트에 대한 기본 간격은 5분이다(이는 unreachable no에 대한 기본 타임아웃 시간인 40초보다 훨씬 길다).
- kubelet은 lease object 생성하고 10초 마다 업데이트(기본 값) 한다. lease에 대한 업데이트는 no의 .status 업데이트와 독립적으로 수행된다. lease 업데이트가 실패하면 kubelet은 200ms를 시작으로 최대 7s까지의 지수 함수 backoff를 사용해 재시도를 수행한다.

아래는 kubectl get -n kube-node-lease lease/${LEASE_NAME} -o yaml 명령어 출력 결과다:

``` yaml
apiVersion: coordination.k8s.io/v1
kind: Lease
metadata:
  creationTimestamp: "2023-03-19T08:52:42Z"
  name: ip-172-31-100-75.ap-northeast-2.compute.internal
  namespace: kube-node-lease
  ownerReferences:
  - apiVersion: v1
    kind: Node
    name: ip-172-31-100-75.ap-northeast-2.compute.internal
    uid: aadf774b-902a-45bf-ba09-f24b4024bbcb
  resourceVersion: "4442214"
  uid: 3f254bdd-c2dc-44a4-9fd3-1461316b9055
spec:
  holderIdentity: ip-172-31-100-75.ap-northeast-2.compute.internal
  leaseDurationSeconds: 40
  renewTime: "2023-03-27T02:00:40.965759Z"
```

k8s lease 리소스는 no 리소스에 종속되었다.

## Node controller
node controller는 no의 다양한 측면을 관리하는 k8s control plane 구성요소다.

node controller는 no의 생명 주기 동안 여러 역할을 맡는다.

1. no가 등록될 때 CIDR 블락을 할당(CIDR 할당이 활성화 된 경우)한다.
2. 두 번째는 controller의 내부 no 목록을 cloud provider의 사용 가능한 시스템 목록을 참고해 최신 상태로 유지하는 것이다. 클라우드 환경에서 실행할 때 no가 unhealthy 상태가 되면, node controller는 no에 대한 시스템이 이용 가능한지 cloud prider에 확인한다. 이용이 불가할 경우 node controller는 no 목록에서 해당 no를 삭제한다.
3. no의 상태를 모니터링한다. node controller는 다음과 같은 책임이 있다:
    - no가 unreachable 상태가 될 경우, no의 .status 필드의 Ready condition을 업데이트 한다. 이 경우 node controller는 Ready condition을 unknown으로 변경한다.
    - no가 unreachable 상태로 남아있는 경우, unreachable no의 po를 위해 API-initiated eviction API를 트리거한다. 기본적으로 node controller는 Unknown 상태가된 시점부터 첫 eviction 요청까지 5분 동안 기다린다.

기본적으로 node controller는 각 no의 상태를 5초 마다 확인한다. 이 주기는 kube-controller-manager 구성요소의 --node-monitor-period flag를 사용해 설정할 수 있다.

### Rate limits on eviction

## Resource capacity tracking
no object는 node의 리소스 용량에 대한 정보를 추적한다: 예를 들어 이용 가능한 메모리와 CPU 정보. self register no는 등록 시 용량에 대한 정보를 제공한다. 직접 no를 추가할 경우 용량 정보를 설정해야 한다.

k8s scheduler는 no에 실행 중이s po에 대한 충분한 리소스가 있음을 보장한다. scheduler는 no에 존재하는 container의 resource request에 대한 총합이 no의 용량보다 크지 않음을 확인한다. request의 총합은 kubelet에 의해 관리되는 모든 container를 포함하며 container runtime을 통해 직접 실행된 container와 kubelet의 제어외의 프로세스는 제외한다.

**Note**: non-po 프로세스에 대한 자원을 미리 예약하기 원할 경우 [reserve resources for system daemons](https://kubernetes.io/docs/tasks/administer-cluster/reserve-compute-resources/#system-reserved) 페이지를 참고한다.

## Node topology
TopologyManager feature gate를 활성화한 경우 kubelet은 리소스 할당 결정을 할 때 topology 힌트를 이용할 수 있다. 관련해 [Control Topology Management Policies on a Node](https://kubernetes.io/docs/tasks/administer-cluster/topology-manager/) 페이지를 참고한다.

## Graceful node shutdown
kubelet은 node system의 shutdown 감지를 시도하고 실행 중인 po를 종료한다. systemd inhibitor lock을 이용해 주어진 시간 동안 node의 종료를 지연시키기 때문에 systemd에 의존한다.

kubelet은 node shutdown 동안 일반적인 [pod termination process](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination)를 보장한다.

graceful node shutdown은 GracefulNodeShutdown feature gate(k8s 1.21부터 기본 활성화)에 의해 제어된다.

기본적으로 shutdownGracePeriod, shutdownGracePeriodCriticalPods 옵션은 0 값으로 설정되어 gracefule node shutdown 기능을 활성화시키지 않는다. 이 기능을 활성화하기 위해 kubelet에 해당 옵션이 0이 아닌 값으로 변경되어야 한다.

graceful shutdown 동안, kubelet은 2개의 phase를 통해 po를 종료한다:


graceful node shutdown 기능은 2개의 KubeletConfiguration 옵션을 통해 설정된다:

- shutdownGracePeriod: node가 shutdown을 지연할 총 시간을 나타낸다. 이는 regular, critical po에 대한 총 po 종료 시간을 나타낸다.
- shtdownGracePeriodCriticalPods: critical pod의 종료에 사용될 시간을 타나낸다. 해당 옵션은 shutdownGracePeriod보다 작아야 한다.

예를 들어, shutdownGracePeriod=30, shtdownGracePeriodCriticalPods=10일 경우, kubelet은 node shutdown을 30초 지연한다. shutdown 동안 20초 (30 - 10)는 normal po를 종료하는 시간으로 예약되며, 이 후 10초는 critical po를 종료하는 시간으로 예약된다.

**Note**: graceful node shutdown 동안 축출된 po는 shutdown으로 마킹된다. kubectl get po 명령어를 사용해 축축된 po가 Terminated 상태임을 확인할 수 있다. 그리고 kubectl describe po 명령어를 사용해 아래 내용을 확인할 수 있다.

```
Reason:         Terminated
Message:        Pod was terminated in response to imminent node shutdown.
```

## Non Graceful node shutdown

### Pod Priority based graceful node shutdown

## Swap memory management