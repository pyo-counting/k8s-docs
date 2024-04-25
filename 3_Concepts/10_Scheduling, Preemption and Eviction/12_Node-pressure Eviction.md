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

### Eviction thresholds
## Eviction monitoring interval
Node conditions
Node condition oscillation
Reclaiming node level resources
Pod selection for kubelet eviction
Minimum eviction reclaim
Node out of memory behavior
Good practices
Schedulable resources and eviction policies
DaemonSets and node-pressure eviction
Known issues
kubelet may not observe memory pressure right away
active_file memory is not considered as available memory