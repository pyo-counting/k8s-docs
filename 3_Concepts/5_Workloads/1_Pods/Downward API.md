## Downward API
k8s와 결합되지 않으면서 container가 자신에 대한 정보를 갖는 것을 유용할 수 있다. downward API를 사용하면 k8s client 또는 API server를 사용하지 않고 클러스터 또는 자기 자신에 대한 정보를 이용할 수 있다.

container 내에 po, container 필드를 노출하는 2가지 방법이 있다.

- 환경 변수
- downwardAPI volume 내 파일

이 두가 방식을 downward API라고 부른다.

## Available fields
downward API에 사용 가능한 필드를 설명한다.

po 레벨이 필드는 `fieldRef`, container 레벨 필드는 `resourceFieldRef`로 명시한다.

### Information available via `fieldRef`
po 레벨 필드 대부분은 환경 변수 또는 downwardAPI volume으로 container에 제공할 수 있다.
두 방식을 모두 사용 가능한 필드는 아래와 같다:

- `metadata.name`
- `metadata.namespace`
- `metadata.uid`
- `metadata.annotations['<KEY>']`
- `metadata.labels['<KEY>']`
- `spec.serviceAccountName`
- `spec.nodeName`
- `status.hostIP`
- `status.podIP`

아래 필드는 환경 변수로 제공할 수 없으며 downwardAPI volume으로만 제공할 수 있다.

- `metadata.labels`
- `metadata.annotations`

### Information available via `resourceFieldRef`
container 레벨 필드는 CPU, memory와 같은 리소스의 request, limit 정보를 제공한다.

- `resource: limits.cpu`
- `resource: requests.cpu`
- `resource: limits.memory`
- `resource: requests.memory`
- `resource: limits.hugepages-*`
- `resource: requests.hugepages-*`
- `resource: limits.ephemeral-storage`
- `resource: requests.ephemeral-storage`

#### Fallback information for resource limits  
container에 CPU. memory 제한이 없는 경우 downward API를 사용할 때, kubelet은 [node allocatable](https://kubernetes.io/docs/tasks/administer-cluster/reserve-compute-resources/#node-allocatable) 계산을 통해 최대 CPU, memory 값을 기본값으로 노출한다.