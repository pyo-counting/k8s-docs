po는 다른 po와의 상대적 중요성을 나타내기 위한 priority가 있다. po가 스케줄링될 수 없는 경우 kube-scheduler는 pending po보다 더 낮은 priority를 갖는 po를 preempt(evict)를 시도한다.

> **Warning**:  
> priority가 높은 po의 생성을 막기 위해 관리자는 ResourceQuota 리소스를 사용할 수 있다.

## How to use priority and preemption
priority, preemption 사용 방법은 다음과 같다.
1. pc를 생성한다.
2. po의 `.spec.priorityClassName` 필드를 이용해 생성한 pc를 사용한다.

> **Note**:  
> k8s는 기본적으로 system-cluster-critical, system-node-critical pc를 제공한다. These are common classes and are used to [ensure that critical components are always scheduled first](https://kubernetes.io/docs/tasks/administer-cluster/guaranteed-scheduling-critical-addon-pods/).

## PriorityClass
pc는 non-namespaced 리소스다. 우선 순위는 `.value` 필드에 정수 값을 설정한다. 값이 높을수록 우선 순위가 높다. 이름은 `system-`으로 시작하면 안된다.

`.value` 필드는 1,000,000,000(1 billion) 이하의 32 bit 정수 값을 가질 수 있다. 즉 우선 순위 값은 -2147483648 <= 우선 순위 <= 1000000000 사이의 값을 갖는다. 더 큰 값은 system po를 위한 내장 pc를 위해 예약됐다.

pc는 `.globalDefault`, `.description` optional 필드를 갖는다.
- `.globalDefault`: `.spec.priorityClassName` 필드를 사용하지 않은 po에 사용될 기본 pc를 나타낸다. 만약 cluster에 기본 pc가 없는 경우 해당 po들은 우선 순위 값 0을 갖는다.
- `.description`: pc에 대한 설명

### Notes about PodPriority and existing clusters
- If you upgrade an existing cluster without this feature, the priority of your existing Pods is effectively zero.
- cluster에 기본 pc를 생성한 경우 기존 po의 우선 순위는 변경되지 않으며 새로 생성되는 po에만 영향을 미친다.
- 실행 중인 po들이 참조하는 pc를 삭제하는 경우, po가 참조하는 pc의 이름이 변경되지는 않지만 새로 생성되는 po는 삭제된 pc의 이름을 참조할 수 없다.

### Example PriorityClass
``` yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
description: "This priority class should be used for XYZ service pods only."
```

## Non-preempting PriorityClass

### Example Non-preempting PriorityClass
## Pod priority
### Effect of Pod priority on scheduling order
## Preemption
### User exposed information
### Limitations of preemption
## Troubleshooting
### Pods are preempted unnecessarily
### Pods are preempted, but the preemptor is not scheduled
### Higher priority Pods are preempted before lower priority pods
## Interactions between Pod priority and quality of service