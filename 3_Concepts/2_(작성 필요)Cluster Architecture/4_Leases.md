분산 시스템에서는 공유 리소스를 lock하고 집합 구성원 간의 활동을 조정하는 메커니즘을 제공하는 lease가 필요한 경우가 있다. k8s에서 lease 개념은 no의 heartbeat, 구성 요소 레벨에서의 리더 선출과 같은 시스템에 중요한 기능에 사용되는 coordination.k8s.io API Group의 Lease 리소스로 표시된다.

## Node heartbeats
k8s는 k8s API server로의 kubelet no heartbeat를 위해 Lease API를 사용한다. 모든 no에 대해 kube-node-lease ns에 동일한 이름을 갖는 Lease object가 있다. 기본적으로 모든 kubelet heartbeat는 Lease 오브젝트의 .spec.renewTime 필드를 업데이트하는 요청이다. k8s control plane은 no의 가용성을 결정하는데 해당 필드를 사용한다.

## Leader election
또한 k8s는 구성요소의 1개의 인스턴스만 실행되도록 보장한다. 이는 HA 구성에서 kube-controller-manager, kube-scheduler와 같은 control plane의 구성요소에서 사용하며 구성 요소의 인스턴스 하나만 활성화되고 다른 인스턴스는 대기상태여야 한다.

## API server identity


## Workloads
사용자는 workload에 대한 lease를 정의할 수도 있다. 예를 들어 peer가 아닌 leader가 작업을 수행하는 커스텀 controller를 실행할 수 있다. lease를 정의함으로써 controller replica는 coordination에 대한 k8s API를 이용해 leader를 선출할 수 있다. lease를 사용하기 위해서는 구성요소와 연결될 수 있도록 관련된 이름을 사용하는 것이 best practice다. 예를 들어, Foo라는 구성요소가 있는 경우 example-foo 이름을 갖는 lease를 사용한다.

cluster 관리자 또는 다른 엔드 유저가 구성요소의 여러 인스턴스를 배포할 수 있는 경우 이름 접두사를 선택하고 메커니즘(deploy 이름에 대한 해싱)을 선택해 lease의 이름 충돌을 피해야한다.

동일한 결과를 얻어낼 수 있는 경우 다른 방식을 사용할 수도 있다.