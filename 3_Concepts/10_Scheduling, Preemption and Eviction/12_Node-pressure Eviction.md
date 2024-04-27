node-pressure eviction은 kubelet이 no의 리소스를 회수하기 위해 po를 사전에 종료하는 프로세스다.

kubelet은 cluster의 no에 있는 메모리, 디스크 공간, 파일 시스템 inode와 같은 리소스를 모니터링한다. 이러한 리소스 중 하나 이상이 특정 소비 레벨에 도달하면 kubelet은 no에 있는 하나 이상의 po를 선제적으로 실패시켜 리소스를 회수하고 기아(starvation)를 방지할 수 있다.

node-pressure eviction 동안 kubelet은 선택한 po의 `.status.phase`를 Failed로 설정하고 po를 종료시킨다.

node-pressure eviction는 API-initiated eviction과 다르다.

kubelet은 설정된 pdb 또는 po의 `.spec.terminationGracePeriodSeconds`를 존중하지 않는다. 만약 soft eviction threshold을 사용하는 경우 kubelet은 `.evictionMaxPodGracePeriod` 설정 값을 존중한다. 만약 hard eviction threshold를 사용하면 kubelet은 0s 값(즉시 종료)을 사용한다.

## Self healing behavior
kubelet은 사용자의 po를 종료하기 전에 reclaim node-level resources를 시도한다. 예를 들어 디스크 리소스가 부족할 때 사용하지 않는 container image를 삭제한다.

po를 workload management object(예를 들어 sts, deploy)가 관리하는 경우 control plane(kube-controller-manager)은 새로운 po를 다시 생성한다.

## Self healing for static pods
resource pressure를 겪는 no에서 [static pod](https://kubernetes.io/docs/concepts/workloads/pods/#static-pods)를 실행 중인 경우 kubelet은 static po도 eviction할 수 있다. static po는 해당 no에서 실행되어야 하는 목적이 있기 때문에 kubelet은 다시 static po를 생성하려고 시도한다.

이 때 kubelet은 static po의 priority를 고려한다. static po의 manifest가 낮은 priority를 정의하고, cluster의 control plane에 높은 priority po가 정의되어 있으면 kubelet은 static po를 위한 공간을 확보하지 못할 수도 있다.

## Eviction signals and thresholds
kubelet은 아래와 같은 다양한 파라미터를 사용해 eviction 결정을 내린다.
- Eviction signals
- Eviction thresholds
- Monitoring intervals

### Eviction signals
eviction signal은 특정 시점의 리소스 상태를 나타낸다. kubelet은 no에서 필요로하는 최소한의 available 리소스를 나타내는 eviction threshold과 eviction signal을 비교해 eviction을 결정한다.

linux에서 kubelet은 아래 eviction signal을 사용한다.
| Eviction Signal    | Description                                                                     |
|--------------------|---------------------------------------------------------------------------------|
| memory.available   | memory.available := node.status.capacity[memory] - node.stats.memory.workingSet |
| nodefs.available   | nodefs.available := node.stats.fs.available                                     |
| nodefs.inodesFree  | nodefs.inodesFree := node.stats.fs.inodesFree                                   |
| imagefs.available  | imagefs.available := node.stats.runtime.imagefs.available                       |
| imagefs.inodesFree | imagefs.inodesFree := node.stats.runtime.imagefs.inodesFree                     |
| pid.available      | pid.available := node.stats.rlimit.maxpid - node.stats.rlimit.curproc           |

signal은 퍼센티지 또는 값을 지원한다. kubelet은 siganl과 관련있는 리소스의 총 capacity를 기준으로 퍼센티지 값을 계산한다.

`memory.available`의 값은 `free -m`와 같은 도구 대신 cgroupfs의 값을 사용한다. `free -m`은 container에 대해 동작하지 않으며, and if users use the [node allocatable](https://kubernetes.io/docs/tasks/administer-cluster/reserve-compute-resources/#node-allocatable) feature, out of resource decisions are made local to the end user Pod part of the cgroup hierarchy as well as the root node. [script](), [cgroupv2 script]()는 kubelet이 `memory.available`을 계산하는 과정을 보여준다. kubelet은 inactive_file(inactive LRU 목록에서 파일 백업 메모리 byte)을 계산에서 제외한다.

filesystem과 관련해 kubelet은 두 식별자만 사용한다.
1. `nodefs.*`: no의 main filesystem을 나타내며 local disk volume, emptyDir volume, log storage 등에 사용된다. 예를 들어 `nodefs`는 `/var/lib/kubelet/`을 포함한다.
2. `imagefs.*`: container image, writable layer를 저장하는 데 사용하는 container runtime의 filesystem

kubelet은 위와 같은 filesystem을 자동으로 탐색하며 다른 로컬 filesystem은 무시한다.

일부 gc 기능은 eviction으로 인해 deprecated됐다.
| Existing Flag                           | Rationale                                                          |
|-----------------------------------------|--------------------------------------------------------------------|
| --maximum-dead-containers               | deprecated once old logs are stored outside of container's context |
| --maximum-dead-containers-per-container | deprecated once old logs are stored outside of container's context |
| --minimum-container-ttl-duration        | deprecated once old logs are stored outside of container's context |

### Eviction thresholds
사용자는 eviction decision에 사용되는 soft/hard eviction threshold을 설정할 수 있다.

eviction threshold은 `[eviction-signal][operator][quantity]` 형태를 갖는다.
- `eviction-signal`: 사용할 eviction siganl
- `operator`: <(~보다 작은)과 같은 관계 연산자
- `quantity`: 1Gi와 같은 eviction threshold을 나타내는 값. 퍼센티지는 %을 사용한다.

예를 들어 총 10GiB를 갖는 no에 대해 available memory가 1GiB 아래일 경우 eviction을 트리거하고 싶다면, eviction threshold를 `memory.available<10%` 또는 `memory.available<1Gi`와 같이 사용할 수 있다.

#### Soft eviction thresholds
soft eviction threshold은 `.evictionSoftGracePeriod`이 초과되기 전까지는 kubelet이 pod를 eviction하지 않는다. 해당 설정을 지정하지 않으면 kubelet 실행 시 오류를 반환한다.

만약 `.evictionSoftGracePeriod` 동안 `.evictionSoft`를 유지하면 `.evictionMaxPodGracePeriod`은 soft eviction 시 po의 eviction에서 사용하는 po의 termination grace period로 사용된다. 해당 설정을 하지 않으면 kubelet은 grace termination 없이 즉시 eviction을 수행한다.

아래는 soft eviction threshold와 관련있는 설정이다.
- `.evictionSoft`: soft eviction threshold. 예: `memory.available<1.5Gi`
- `.evictionSoftGracePeriod`: po의 eviction을 수행하기 전에 eviction threshold가 유지되어야 하는 시간. 예: memory.available=1m30s
- `.evictionMaxPodGracePeriod`: soft eviction 동안 po를 종료에 사용될 수 있는 최대 grace termination period.

#### Hard eviction thresholds
hard eviction threshold은 grace period와 같은 설정이 없다. hard eviction threshold를 충족하면 kubelet은 즉시 po를 eviction한다.

`.evictionHard`를 사용해 hard eviction threshold를 설정할 수 있다.

아래는 kubelet의 기본 hard eviction threshold 설정이다.
- `memory.available<100Mi`
- `nodefs.available<10%`
- `imagefs.available<15%`
- `nodefs.inodesFree<5%` (Linux nodes)
- `imagefs.inodesFree<5%` (Linux nodes)

위 hard eviction threshold 기본 값은 `.evictionHard` 값을 변경하지 않을 경우에만 설정된다. 만약 변경하면 기본 값은 상속되지 않으며 모두 0으로 설정된다. 그렇기 때문에 모든 일부 값만 수정하길 원할 경우 다른 기본 threshold는 모두 설정해야 한다.

## Eviction monitoring interval
kubelet은 `housekeeping-interval`에 설정된 10s를 기본 값으로 eviction threshold을 평가한다.

위 설정의 경우 kubelet에 사용자가 설정할 수 없는 것 같다(flag, 설정 파일 내 관련 변수 찾을 수 없음).

## Node conditions
kubelet은 설정된 grace period와 관련 없이 hard 또는 soft eviction threshold가 충족되어 no가 pressure을 받고 있는지 보여주기 위해 [node conditions](https://kubernetes.io/docs/reference/node/node-status/#condition)를 보고한다.

kubelet은 eviction signal과 no의 condition을 매핑해서 표현한다.
| Node Condition | Eviction Signal                                                               | Description                                                                                                                  |
|----------------|-------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| MemoryPressure | memory.available                                                              | Available memory on the node has satisfied an eviction threshold                                                             |
| DiskPressure   | nodefs.available, nodefs.inodesFree, imagefs.available, or imagefs.inodesFree | Available disk space and inodes on either the node's root filesystem or image filesystem has satisfied an eviction threshold |
| PIDPressure    | pid.available                                                                 | Available processes identifiers on the (Linux) node has fallen below an eviction threshold                                   |

control plane은 no condition과 [taint](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/#taint-nodes-by-condition)를 매핑한다.

kubelet은 `.nodeStatusUpdateFrequency`(기본 값 10s)를 사용해 no condition을 업데이트한다.

### Node condition oscillation
no에 정의된 grace period 동안 지속적으로 유지되지 않고 soft eviction threshold 기준으로 위 아래 진동할 수 있다. 즉 no condition이 true와 false 사이에서 계속 전환되며 원치 않는 eviction이 발생할 수도 있다.

이를 막기 위해 kubelet의 `.evictionPressureTransitionPeriod` 사용할 수 있다. 해당 값은 kubelet이 no condition을 변경하기 전에 기다려야 하는 시간이다. 기본 값은 5m이다.

### Reclaiming node level resources
kubelet은 사용자의 po를 eviction하기 전에 no의 리소스를 회수를 시도한다.

`DiskPressure` no condition이 true라면 kubelet은 no의 filesystem을 기반으로 리소스를 회수한다.

#### With `imagefs`
container runtime이 사용할 imagefs filesystem이 지정된 경우 kubelet은 아래 동작을 수행한다.
- `nodefs` filesystem이 eviction threshold를 충족하면 kubelet은 종료된 po, container를 gc한다.
- `imagefs` filesystem이 eviction threshold를 충족하면 kubelet은 사용되지 않는 image를 모두 삭제한다.

#### Without `imagefs`
`nodefs` filesystem만 있는 경우 kubelet은 아래 동작을 수행한다.
1. kubelet은 종료된 po, container를 gc한다.
2. kubelet은 사용되지 않는 image를 모두 삭제한다.

### Pod selection for kubelet eviction
no 리소스 회수를 수행했음에도 threshold를 넘는다면 kubelet은 사용자 po에 대한 eviction을 수행한다.

kubelet은 다음 파라미터를 사용하여 po eviction 순서를 결정한다.
1. po의 리소스 사용량이 request를 초과하는지 여부
2. [Pod Priority](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/)
3. po의 리소스 request 대비 사용량

결과적으로 kubelet은 아래 순서를 사용해 순위를 정하고 po릉 eviction한다.
1.
2.

#### With `imagefs`
#### Without `imagefs`

Minimum eviction reclaim
Node out of memory behavior
Good practices
Schedulable resources and eviction policies
DaemonSets and node-pressure eviction
Known issues
kubelet may not observe memory pressure right away
active_file memory is not considered as available memory