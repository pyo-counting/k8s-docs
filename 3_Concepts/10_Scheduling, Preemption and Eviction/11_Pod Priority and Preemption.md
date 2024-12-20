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
`.preemptionPolicy` 가 `Never`인 pc를 갖는 po는 우선 순위가 낮은 po보다 scheduling queue에 더 앞에 위치하게되지만 다른 po를 preempt할 수 없다. 이러한 non-preempting po는 충분한 resource가 확보될 때까지 scheduling queue에 대기한다. non-preempting po는 다른 po와 마찬가지로 scheduler의 back-off의 대상이 된다. 즉, po의 스케줄링을 시도하지만 불가능한 경우 재시도 간격을 점점 늘리며 이로 인해 우선 순위가 낮은 다른 po가 먼저 스케줄링될 수 있도록 허용된다는 것을 의미한다.

non-preempting po는 우선 순위가 높은 po에 의해 preempt될 수 있다.

`.preemptionPolicy`의 기본 값은 낮은 우선 순위를 갖는 po를 preempt할 수 있는 `PreemptLowerPriority`이다. 만약 `Never`라면 다른 po를 preempt하지 않는다.

non-preempting po의 사용 예시는 data science workload다. 사용자는 다른 workload 보다 우선 순위가 높은 job을 실행할 수 있지만 이미 실행 중인 po를 preempt하지 않길 원한다. non-preempting po는 cluster에 resource가 충분해지면 자연스럽게 스케쥴링 된다.

### Example Non-preempting PriorityClass
``` yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority-nonpreempting
value: 1000000
preemptionPolicy: Never
globalDefault: false
description: "This priority class will not cause other pods to be preempted."
```

## Pod priority
po를 생성할 때 `.spec.priorityClassName` 필드를 통해 pc를 참조함으로써 po의 우선 순위를 명시할 수 있다. priority admission controller는 이 필드를 사용해 실제 우선 순위를 나타내는 정수 값을 `.spec.priority` 필드에 할당한다. 만약 po가 유효하지 않은 pc를 참조하는 경우 po는 reject된다.

아래 yaml은 위 예시에서 생성한 pc를 참조하는 po 예시다. priority admission controller는 high-priority pc를 참조해 `.spec.priority` 필드에 정수 값 1000000을 설정한다.

### Effect of Pod priority on scheduling order
po에 대한 우선 순위가 활성화된 경우 scheduler는 scheduling queue에서 pending po의 우선 순위에 따라(우선 순위가 높은 pending po가 앞에 위치) 정렬를 수행한다. 결과적으로 요구 사항을 만족한다면 우선 순위가 높은 po가 먼저 스케줄링 된다. 만약 스케줄링에 실패할 경우 우선 순위가 낮은 po를 스케줄링하기 위해 시도한다.

## Preemption
po가 생성되면 먼저 스케줄링이 되기 위해 scheduling queue에 대기한다. scheduler는 qeueue에 po를 선택해 no에 스케줄링하기 위해 노력한다. 만약 pending po의 모든 요구 사항을 만족하는 no가 없다면 preemption 로직이 트리거된다. pending po를 P라고 칭하자. preemption 로직은 우선 순위가 P보다 낮은 하나 이상의 po를 제거하면 P를 스케줄링할 수 있는 no를 찾으려고 시도한다. 만약 이러한 no가 발견되면, 우선 순위가 낮은 하나 이상의 po가 해당 노드에서 축출(evicted)된다. po가 제거된 후, P는 해당 no에 스케줄링될 수 있다.

### User exposed information
po P가 no N에서 하나 이상의 po를 선점(preempt)할 때, po P의 `.status.nominatedNodeName` 필드 값은 no N의 이름으로 설정된다. 이 필드는 scheduler가 po P를 위해 예약된 리소스를 추적하는 데 도움을 주며, 사용자에게 cluster에서 발생한 preempt에 대한 정보를 제공한다.

하지만 po P가 반드시 "nominated Node"에 스케줄링되는 것은 아니다. victim po가 preempt된 후 설정에 따라 graceful termination period를 갖는다. scheduler는 victim po의 종료를 기다리는 동안 다른 no에 스케줄링이 가능한 경우, nominated no가 아닌 다른 no에 스케줄링할 수도 있다. 결과적으로 po의 `.status.nominatedNodeName`와 `.spec.nodeName` 필드 값이 다를 수 있다. 그리고 scheduler나 no N에서 po를 preempt 하더라도 po P보다 우선 순위가 더 높은 다른 po가 있을 경우 po P가 아닌 우선 순위가 더 높은 po를 할당할 수도 있다. 이런 경우 scheduler는 po P에 설정됐던 `.status.nominatedNodeName`을 다시 빈 값으로 설정한다. 이를 통해 scheduler는 po P가 다른 no에서 po를 preempt할 수 있도록 허용한다.

### Limitations of preemption
#### Graceful termination of preemption victims
victim po가 preempt될 때 설정에 따라 graceful termination period를 갖는다. 이로 인해 preempt한 시점과 preempt를 야기한 우선 순위가 높은 po가 실제 해당 no에 스케줄링되기까지 시간 차이가 발생한다(물론 scheduler는 계속해서 다른 pending po에 대한 스케줄링을 수행한다). graceful termination period가 만료되고 victim po가 종료되면 해당 po를 스케줄링하기 위해 다시 scheduling queue에 넣는다. 결과적으로 victim po의 preempt 시점과 po가 스케줄링 되는 시점 간 시간 차이가 발생한다. 이 시간 차이를 최소화하기 위해 graceful termination period을 짧은 시간으로 설정할 수 있다.

#### PodDisruptionBudget is supported, but not guaranteed
pdb는 voluntary disruption에 대해 동시에 종료될 수 있는 po의 replica 개수를 제한한다. k8s는 po preempt에 대해 pdb를 지원하지만 완전 보장하지는 않으며 최대 노력(best effort)한다. scheduler는 기본적으로 preempt에 의해 pdb를 보장할 수 있는 po를 찾기위해 노력한다. 하지만 찾지 못할 경우 pdb를 위반하더라도 우선 순위가 낮은 po를 삭제할 수 있다.

#### Inter-Pod affinity on lower-priority Pods

#### Cross node preemption

## Troubleshooting
### Pods are preempted unnecessarily
### Pods are preempted, but the preemptor is not scheduled
### Higher priority Pods are preempted before lower priority pods
## Interactions between Pod priority and quality of service