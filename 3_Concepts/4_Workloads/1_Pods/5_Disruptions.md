고가용성을 제공하기 위한 애플리케이션을 생성하기 위해 po에서 발생할 수 있는 disruption 종류를 이해해야 한다.

뿐만 아니라 클러스터 업그레이드 및 autoscale과 같은 자동 클러스터 작업을 원하는 클러스터 관리자를 위해서도 필요하다.

## Voluntary and involuntary disruptions
po는 누군가(사람 또는 controller)가 삭제하지 않는한 사라지지 않는다. 물론 피할 수 없는 하드웨어 또는 시스템 소프트웨어 오류가 있을 수 있다.

이러한 불가피한 상황을 애플리케이션의 비자발적 중단(involuntary disruption)이라고 부른다. 예를 들어
- a hardware failure of the physical machine backing the node
- cluster administrator deletes VM (instance) by mistake
- cloud provider or hypervisor failure makes VM disappear
- a kernel panic
- the node disappears from the cluster due to cluster network partition
- eviction of a pod due to the node being [out-of-resources](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/).

out-of-resources 이외에는 대부분 사용자에게 익숙할 것이다. 이는 k8s에 특정된 것은 아닌다.

위와 다른 상황을 자발적 중단(voluntary disruption)이라고 부른다. 여기에는 애플리케이션 소유자의 작업, 클라스터 관리자의 작업이 모두 포함된다. 아래는 대표적인 애플리케이션 소유자의 작업이다.
- deleting the deployment or other controller that manages the pod
- updating a deployment's pod template causing a restart
- directly deleting a pod (e.g. by accident)

아래는 클러스터 관리자의 작업이다.
- Draining a node for repair or upgrade.
- Draining a node from a cluster to scale the cluster down (learn about Cluster Autoscaling ).
- Removing a pod from a node to permit something else to fit on that node.

위 작업은 클러스터 관리자가 직접 수행하거나 자동화를 통해 수행하며, 클러스터 호스팅 공급자에 의해서도 수행된다.

클러스터에 자발적인 중단을 일으킬 수 있는 어떤 원인이 있는지 클러스터 관리자에게 문의하거나 클라우드 공급자에게 문의하고, 배포 문서를 참조해서 확인해야 한다. 만약 자발적인 중단을 일으킬 수 있는 원인이 없다면 pdb의 생성을 생략할 수 있다.

> **caution**:  
> 모든 자발적 중단이 pdb에 연관되는 것은 아니다. 예를 들어 deploy, po에 대한 삭제는 pdb를 무시한다.

## Dealing with disruptions
비자발적인 중단으로 인한 영향을 줄이기 위한 몇 가지 방법은 다음과 같다.
- po가 필요로 하는 resource를 요청하는지 확인한다.
- 고가용성이 필요한 경우 애플리케이션의 replica를 운영한다.
- repliace 운영 시 훨씬 더 높은 가용성을 위해 rack 전체(anti-affinity 사용) 또는 zone여러 곳(다중 영역 클러스터를 이용한다면)에 애플리케이션을 분산해야 한다.

자발적 중단의 빈도는 다양하다. 기본적인 k8s 클러스터에서는 자동화된 자발적 중단은 발생하지 않는다(사용자가 지시한 자발적 중단만 발생한다). 그러나 클러스터 관리자 또는 호스팅 공급자가 자발적 중단이 발생할 수 있는 일부 부가 서비스를 운영할 수 있다. 예를 들어 no 소프트웨어의 업데이트를 출시하는 경우 자발적 중단이 발생할 수 있다. 또한 클러스터(no) 오토스케일링의 일부 구현에서는 단편화를 제거하고 no의 효율을 높이는 과정에서 자발적 중단을 야기할 수 있다. 클러스터 관리자 또는 호스팅 공급자는 예측 가능한 자발적 중단 수준에 대해 문서화해야 한다. po spec 안에 PriorityClass와 같은 특정 환경설정 옵션 또한 자발적(그리고 비자발적) 중단을 유발할 수 있다.

## Pod disruption budgets
k8s는 자발적 중단이 자주 발생하는 경우에도 고가용성 애플리케이션을 실행하는 데 도움이 되는 기능을 제공한다.

애플리케이션 소유자는 각 애플리케이션에 대해 pdb를 만들 수 있다. pdb는 자발적 중단으로 일시에 중지되는 replica의 갯수를 제한한다. 예를 들어, quorum-based 애플리케이션이 실행 중인 replica의 수가 필요 quorum 이하가 되지 않도록 한다. 웹 프론트엔드는 부하를 처리하는 replica의 수가 일정 비율 이하로 떨어지지 않도록 보장할 수 있다.

클러스터 관리자와 호스팅 공급자는 직접적으로 po나 deploy를 제거하는 대신 Eviction API로 불리는 pdb을 준수하는 툴을 이용해야 한다.

예를 들어, kubectl drain 명령어를 사용하면 no를 서비스 중단으로 표시할 수 있다. kubectl drain을 실행하면 해당 no의 모든 po eviction을 시도한다. kubectl이 사용자를 대신하여 수행하는 evviction 요청은 일시적으로 거부될 수 있으며 대상 no의 모든 po가 종료되거나 설정 가능한 타임아웃이 도래할 때까지 주기적으로 모든 실패된 요청을 다시 시도한다.

pdb는 애플리케이션이 필요로하는 replica의 수를 기준으로 상대적으로 용인할 수 있는 replica의 수를 지정한다. 예를 들어 `.spec.replicas`의 값이 5인 deploy는 어느 시점에든 5개의 po를 가져야 한다. 만약 해당 deploy의 pdb가 특정 시점에 po를 4개 허용한다면, Eviction API는 한 번에 1개(2개의 po가 아닌)의 po의 자발적인 중단을 허용한다.

po 그룹은 label selector를 사용해서 지정한 애플리케이션으로 구성되며 애플리케이션 controller(deploy, sts 등)를 사용한 것과 같다.

po의 "의도"하는 수는 해당 po를 관리하는 workload resource의 `.spec.replicas`를 기반으로 계산한다. control plane은 po의 `.metadata.ownerReferences`를 검사하여 소유하는 workload resource를 발견한다.

비자발적 중단은 pdb로는 막을 수 없지만 budget은 차감된다.

애플리케이션의 rolling upgrade로 po가 삭제되거나 사용할 수 없는 경우 disruption budget에 영향을 준다. 그러나 workload resource(deploy, sts과 같은)는 rolling upgrade 시 pdb의 제한을 받지 않는다. 대신, 애플리케이션 업데이트 중 실패 처리는 특정 workload resource에 대한 spec에 구성된다.

Eviction API를 사용하여 po를 축출하면, po의 `.spec.terminationGracePeriodSeconds` 필드 값을 준수하여 grafecully terminate된다.

## PodDisruptionBudget example
node-1 ~ node-3이 있다고 가정한다. 그리고 pod-a, pod-b, pod-c라고 불리는 3개의 replica가 있고 pdb와 관련 없는 pod-x po가 있다. 아래는 초기 상태다.
node-1|node-2|node-3
------|------|------
pod-a *available*|pod-b *available*|pod-c *available*|
pod-x *available*||

3개의 po는 deploy에 의해 관리되고 항상 3개의 po중 최소 2개의 po를 유지할 수 있도록 pdb를 설정했다.

이 떄 클러스터 관리자는 커널 버전 버그 해결을 위해 kubectl drain 명령어를 사용해 node-1을 drain한다. 해당 명령어는 pod-a, pod-x를 축축하려고 시도한다. 이는 성공한다. 두 po 모두 동시에 terminating 상태가 된다.
node-1 *draining*|node-2|node-3
------|------|------
pod-a *terminating*|pod-b *available*|pod-c *available*|
pod-x *terminating*||

deploy는 po의 종료를 인지하고 pod-d라는 대체 po를 생성한다. 그리고 어떤 controller에 의해 pod-y도 대체 생성된다.

(Note: sts의 경우 pod-a는 대신 pod-0과 같은 이름을 가졌을 것이며 동일한 이름의 대체 po가 생성되기 전에 완벽히 종료되어야 한다.)

node-1 *draining*|node-2|node-3
------|------|------
pod-a *terminating*|pod-b *available*|pod-c *available*|
pod-x *terminating*|pod-d *starting*|pod-y

아래는 node-1이 완벽히 drain된 이후 상태다.
node-1 *drained*|node-2|node-3
------|------|------
||pod-b *available*|pod-c *available*|
||pod-d *starting*|pod-y

위 상황에서 클러스터 관리자가 node-2 또는 node-3를 drain하려고 할 때 실패할 것이다. 왜냐하면 pdb는 적어도 2개의 replica가 필요하기 때문이다. pod-d가 정상 실행 상태가 됐을 때 클러스터 관리자가 node-2를 drain한다. drain 명령어는 어떤 순서를 기준으로 pod-b부터 축출하려고 시도한다. pod-b에 대한 evict는 성공하지만 pod-d에 대한 축축을 실패한다. 왜냐하면 최소 2개의 po를 만족하지 않기 때문이다.

아래는 pod-b의 종료에 따른 대체 po를 생성하려는 cluster의 상태다. 하지만 cluster 내 해당 po를 스케줄링 할 수 있는 no가 없어 pending 상태가 됐다. 이 떄 cluster는 아래 상태에서 멈추게 된다.
node-1 *drained*|node-2|node-3|*no node*
------|------|------|------
||pod-b *available*|pod-c *available*|pod-e *pending*
||pod-d *starting*|pod-y|

이 상황에서 클러스터 관리자는 정상 진행을 위해 no를 추가해야 한다.

## Pod disruption conditions

## Separating Cluster Owner and Application Owner Roles
보통 클러스터 매니저와 애플리케이션 소유자는 서로에 대한 지식이 부족한 별도의 역할로 생각하는 것이 유용하다. 이와 같은 책임의 분리는 다음의 시나리오에서 타당할 수 있다.
- k8s 클러스터를 공유하는 애플리케이션 팀이 많고, 자연스럽게 역할이 나누어진 경우
- third-party tool 또는 서비스를 이용해서 클러스터 관리를 자동화 하는 경우

pdb은 역할 분리에 따라 역할에 맞는 인터페이스를 제공한다.

만약 조직에 역할 분리에 따른 책임의 분리가 없다면 pdb을 사용할 필요가 없다.

## How to perform Disruptive Actions on your Cluster
만약 클러스터 관리자라면, 그리고 클러스터 전체 no에 no 또는 시스템 소프트웨어 업그레이드와 같은 중단이 발생할 수 있는 작업을 수행하는 경우 다음과 같은 옵션을 선택한다.

- 업그레이드 하는 동안 다운타임을 허용한다.
- 다른 레플리카 클러스터로 장애조치를 한다.
    - 다운타임은 없지만, no 복제본과 전환 작업을 조정하기 위한 인력 비용이 많이 발생할 수 있다.
- pdb를 이용해서 애플리케이션의 중단에 영향을 받지않도록 한다.
    - 다운타임 없음
    - 최소한의 리소스 중복
    - 클러스터 관리의 자동화 확대 적용
    - 내결함성이 있는 애플리케이션의 작성은 까다롭지만 자발적 중단를 허용하는 작업의 대부분은 오토스케일링과 비자발적 중단를 지원하는 작업과 겹친다.
