po는 k8s에서 생성 및 관리할 수 있는 가장 작은 단위의 배포 가능한 컴퓨팅 단위다.

po는 공유 storage, 네트워크 리소스 및 실행 할 container에 대한 명세를 갖는 container의 그룹이다. po의 컨텐츠는 항상 같은 공간에 있으며 공유 context에서 실행된다.

po는 애플리케이션 container 뿐만 아니라 po 실행 과정에서 동작하는 init container도 포함할 수 있다. 뿐만 아니라 디버깅을 위한 ephemeral container를 포함할 수 있다.

## What is a Pod?
po의 공유 context는 리눅스 namespace, cgroup 및 잠재적인 격리 요소의 집합이다. po context 안에서 각 애플리에키션은 추가 격리가 적용될 수도 있다.

docker와 비교했을 때 po는 공유 namespace, 공유 filesystem volume를 갖는 container의 집합과 유사하다.

## Using Pods
보통 po는 직접 생성하지 않고 workload resource를 사용한다.

### Workload resources for managing pods
보통 po는 싱글톤이더라도 직접 생성 및 관리하지 않는다. 대신 deploy, job 또는 po가 상태를 추적해야할 경우 sts를 사용해 생성 및 관리한다.

po는 일반적으로 두 가지 방식으로 사용된다:

- 단일 container를 실행. "one-container-per-Pod" 모델은 k8s에서 가장 보편적인 사용 사례다.
- 함께 동작해야 하는 여러 container를 실행하는 po. po는 리소스를 공유하면서 밀접하게 연관있는 container들을 캡슐화할 수 있다.

**Note**: 한 po에 여러 container를 그룹화하는 것은 고레벨의 케이스이다.

### How Pods manage multiple containers
po의 container들은 클러스터 내 동일한 물리 또는 가상 시스템에 배치된다. container들은 리소스, dependency를 공유할 수 있다.

po는 기본적으로 구성 container에 network, storage 두 가지 공유 리소스를 제공한다.

## Working with Pods
k8s 환경에서는 싱글톤 po라도 직접 생성할 일은 거의 없다. 이는 po가 일시적이고 일회용의 목적으로 설계되었기 때문이다.

**Note**: po 내 container를 재시작하는 것과 po를 재시작하는 것과 혼동하면 안된다 po는 프로세스가 아니며 container를 구동하기 위한 환경이다. po는 삭제되기 전까지 지속된다.

po의 이름은 DNS subdomanin name 규칙을 따라야 한다.

### Pods and controllers
여러 po 생성 및 관리를 위해 workload resource를 사용할 수 있다. resource의 controller는 po의 replication, roll out, 실패 상황에서의 치유를 처리한다. 예를 들어 no가 실패되면 controller가 해당 no의 po 동작이 멈춤을 인지하고 대체 po를 생성한다. 그리고 scheduler는 정상 no에 대체 po를 스케쥴링한다.

아래는 1개 이상의 po를 관리하는 workload resource 예시다:

- deploy
- sts
- ds

### Pod templates
workload resource들의 controller는 po template으로부터 po을 생성하고 관리한다.

po template을 수정하는 것은 이미 존재하는 po에는 직접적인 영향을 주지 않는다. workload resource의 po template을 변경한다면 해당 resource는 업데이트된 template을 이용해 대체 po를 생성해야 한다.

예를 들어 sts controller는 각 sts의 현재 po template과 실행 중인 po가 일치되도록 보장한다. 각 workload resource마다 po template 변경 시 이를 다루는 동작에 대한 규칙을 각각 구현한다.

각 no의 kublete은 po template의 세부 사항이나 업데이트를 직접 관리 및 관찰하지 않는다.

## Pod update and replacement
k8s는 po를 workload resource 없이 관리하는 것을 제한하지 않는다. 동작중인 po의 일부 필드를 업데이트하는 것이 가능하다. 하지만 patch, replace와 같은 명령어에는 제한이 있다:

- 대부분의 po medata는 불변이다. 예를 들어 namespace, name, uid, creationTimestamp 필드를 변경할 수 없다; generation 필드는 고유하며 현재 값보다 증가한 값을 업데이트 가능하다.
- metadata.deletionTimestamp가 설정되었다면 metadata.finalizers 목록에 새로운 항목을 추가할 수 없다.
- spec.containers[*].image, spec.initContainers[*].image, spec.activeDeadlineSeconds, spec.tolerations 외 po의 업데이트는 변경이 불가하다. spec.tolerations에 대해서는 새로운 항목을 추가할 수 있다.
- spec.activeDeadlineSeconds 필드를 업데이트할 때 두 업데이트 타입이 가능하다:
  1. 할당되지 않은 필드에 대해 양수로 설정
  2. 필드가 양수 값일 경우 더 작은 양수로 업데이트

## Resource sharing and communication
po내 container 간에는 데이터 공유 및 통신이 가능하다.

### Storage in Pods
po에 shared storage volume을 명시할 수 있다. po의 모든 container는 shared volume에 접근 할 수 있으며 데이터를 공유할 수 있다. 또한 container 재시작 시에도 데이터가 유지될 수 있도록 volume을 사용할 수 있다.

### Pod networking
각 po에는 고유한 ip가 할당된다. po의 모든 container는 네트워크 ip주소, port를 포함하는 네트워크 namespace를 공유한다. po에 속한 container는 서로 localhost를 이용해 통신할 수 있다. container가 po 외부와 통신할 때 공유 네트워크 리소스를 어떻게 이용할지 조정해야한다. 또한 po 내 container끼리 SystemV semaphores, POSIX shared memory와 같은 표준 IPC를 이용해 통신할 수 있다. container가 다른 po의 container와 통신하기 위해서는 ip를 이용한 통신만 가능하다(OS 수준의 IPC를 이용하기 위해서는 설정 필요).

container의 hostname은 po의 이름으로 설정된다.

## Privileged mode for containers
리눅스 환경에서 container에 privileged 옵션을 사용해 privileged mode를 사용할 수 있다.

**Note**:container runtime에서 privileged container 개념을 지원해야 한다.

## Static Pods
static po는 API server의 관찰 없이 특정 노드의 kubelet daemon에 의해 관리된다. deploy와 같이 대부분의 po는 control plane에 의해 관리되는 반면 static po는 kubelet에 직접 관리한다.

static po는 항상 특정 node의 kubelet에 한정된다. static po의 주된 용도는 자체 호스팅 control plane을 실행하는 것이다: 즉, kubelet을 사용해 개별 control plane 구성 요소를 감독한다.

kubelet은 각 static po에 대해 k8s API server에 mirror po(kubelet에 의해 관리되는 static po를 추적하는 object)를 자동으로 생성한다. 이는 no에 실행되는 po가 API server에서 볼 수 있음을 의미하지만 API server를 통해 제어는 하지 못한다.

**Note**: static po의 .spec에서는 다른 API object를 참조할 수 없다(예를 들어 sa, cm, secret 등).

## Container probes
probe는 kubelet에 의해 주기적으로 container를 대상으로 수행된다. kubelet은 다른 유형의 진단을 수행할 수 있다:

- ExecAction (container runtime의 도움으로 수행)
- TCPSocketAction (kubelet에 의해 직접 수행)
- HTTPGetAction (kubelet에 의해 직접 수행)