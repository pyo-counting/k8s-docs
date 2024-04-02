k8s에서 no의 status는 k8s cluster를 관리하는 데 매우 중요한 요소다.

## Node status fields
no의 status는 아래 정보를 포함한다:

- Addresses
- Conditions
- Capacity and Allocatable
- Info

no의 status 및 기타 정보를 조회하기 위해 kubectl 명령어를 사용할 수 있다:

``` bash
kubectl describe node <insert-node-name-here>
```

출력의 각 상세 정보는 아래에서 설명한다. 

아래는 kubectl get no -o yaml 명령어 출력 결과 중 status 부분이다. 아래는 minikube에서의 no `.status` 정보다.
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

아래는 EKS에서의 no `.status` 정보다.
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

## Addresses
`.status.addresses` 필드는 cloud provider 또는 bare metal 설정에 따라 다르다.

- `HostName`: no의 kernel에서 제공하는 hostname. kubelet의 `--hostname-override` 파라미터를 사용해 덮어쓸 수 있다.
- `ExternalIP`: 클러스터 외부에서 라우팅 가능한 no의 IP 주소
- `InternalIP`: 클러스터 내부에서 라우팅할 수 있는 no의 IP 주소

## Conditions
`.status.conditions` 필드는 running no의 상태를 설명한다. conditions type의 종류는 다음과 같다:
|Node Condition    |Description|
|------------------|-----------|
|Ready             |`True`: no가 healthy하고 po를 수용할 수 있는 상태<br>`False`: no가 healthy하지 않고, po를 수용할 수 없는 상태<br>`Unknown`: node controller가 node-monitor-grace-period(기본 값 40s) 설정 값 동안 no의 상태를 알 수 없는 경우|
|DiskPressure      |`True`: disk 크기에 대한 pressure가 있는 경우(disk 용량이 부족할 경우)<br>`False`: 반대의 경우|
|MemoryPressure    |`True`: no 메모리에 대한 pressure가 있는 경우(no의 메모리가 부족할 경우)<br>`False`: 반대의 경우|
|PIDPressure       |`True`: 프로세스에 대한 pressure이 있는 경우(no에 너무 많은 프로세스가 있는 경우)<br>`False`: 반대의 경우|
|NetworkUnavailable|`True`: no의 네트워크가 옳바르게 설정되지 않은 경우<br>`False`: 반대의 경우|

> **Note**:  
> kubectl 명령어를 사용해 cordoned no의 상세 정보를 조회하는 경우, `.status.conditions` 필드에 SchedulingDisabled를 포함한다. SchedulingDisabled은 k8s API에 정의된 Condition이 아니다. 대신 cordoned no는 spec에 Unschedulable로 표시된다.

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

no에 문제가 발생하면 k8s control plance은 no에 영향을 미치는 `.status.conditions`과 일치하는 taint를 자동으로 생성한다. 예를 들어 Ready condtion의 상태가 kube-controller-manager의 NodeMonitorGracePeriod(기본 값 40초)보다 오랫동안 Unknown 또는 False로 유지되는 경우에 해당한다. Unknown 상태면 `node.kubernetes.io/unreachable` taint, False 상태면 `node.kubernetes.io/not-ready` taint가 no에 추가된다.

이러한 taint는 pending po에 대해 kube-scheduler가 po를 no에 할당할 때 no의 taint를 고려한다. 스케쥴링된 기존 po는 NoExecute taint의 영향으로 퇴출될 수도 있다. po는 no에서 계속 실행될 수 있도록 toleration을 포함할 수도 있다.

## Capacity and Allocatable
`.status.allocatable`, `.status.capacity` 필드는 no의 이용 가능한 리소스 정보를 제공한다: CPU, memory, no에 스케줄링 가능한 최대 po 개수

`.status.capacity` 필드는 no가 가진 총 resource의 크기를 나타낸다. `.status.allocatable` 필드는 일반 po에서 사용할 수 있는 no의 리소스 크기를 나타낸다.

You may read more about capacity and allocatable resources while learning how to reserve compute resources on a Node.

## Info
`.status.nodeInfo` 필드는 커널 버전, k8s 버전(kubelet, kube-proxy 버전), container runtime 상세 사항, no가 사용하는 OS와 같은 일반적인 정보를 제공한다. kubelet은 no로부터 이러한 정보를 수집하며 k8s API로 제공한다.

## Heartbeats
no가 전송하는 heartbeat를 통해 k8s cluster에서 사용 가능한 no를 식별할 수있으며 failure가 감지되면 이에 대한 조치를 취한다.

no는 2가지 heartbeat 방식을 사용한다:
- no의 `.status` 필드를 업데이트한다.
- kube-node-lease ns의 lease object. 각 no에 대해 lease object를 갖는다.

no의 `.status`를 업데이트하는 것에 비해 lease resource는 더 간단하다. heartbeat를 위해 lease를 사용하는 것은 큰 클러스터의 경우 성능에 영향을 줄여준다.

kubelet은 no의 `.status`를 생성 및 업데이트, lease object를 업데이트해야 하는 책임이 있다.
- kubelet은 상태가 변경되거나 설정 간격 동안 업데이트가 없는 경우 no의 `.status`를 업데이트 한다. `.status` 업데이트에 대한 기본 간격은 5분이다(이는 unreachable no에 대한 기본 타임아웃 시간인 40초보다 훨씬 길다).
- kubelet은 lease object 생성하고 10초 마다 업데이트(기본 값) 한다. lease에 대한 업데이트는 no의 `.status` 업데이트와 독립적으로 수행된다. lease 업데이트가 실패하면 kubelet은 200ms를 시작으로 최대 7s까지의 지수 함수 backoff를 사용해 재시도를 수행한다.

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