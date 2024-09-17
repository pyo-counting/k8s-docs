po를 정의할 때 container가 필요로하는 리소스에 대해 명시할 수 있다. 가장 일반적인 리소스 유형은 CPU와 메모리다.

po 내에서 container에 대한 리소스 `request`를 지정하면 kube-scheduler는 이 정보를 사용해 po가 배치될 no를 결정한다. container에 대한 리소스 `limit`을 지정하면, kubelet은 실행 중인 container가 설정한 제한보다 많은 리소스를 사용할 수 없도록 해당 `limit`을 적용한다. 또한 kubelet은 container가 사용할 수 있도록 해당 시스템 리소스의 최소 `request`를 예약한다.

## Requests and limits
po가 실행 중인 no에 사용 가능한 리소스가 충분하면 container가 해당 리소스에 지정한 `request` 보다 더 많은 리소스를 사용할 수 있도록 허용된다. 그러나 container는 리소스 limit 보다 더 많은 리소스를 사용할 수는 없다.

Limits can be implemented either reactively (the system intervenes once it sees a violation) or by enforcement (the system prevents the container from ever exceeding the limit). Different runtimes can have different ways to implement the same restrictions.

**Note**: 리소스에 대한 `request`를 지정하지 않고 `limit`만 지정하는 경우, 리소스에 대한 `request` 기본 값을 설정하는 admission이 없는 경우 k8s는 `limit` 값을 `request` 값으로 사용한다.

## Resource types
CPU, 메모리는 각각의 리소스 타입이다. 리소스 타입은 기본 단위룰 갖는다. 리눅스의 경우 huge page 리소스를 명시할 수 있다. huge page는 kernel이 기본 page 크기 보다 훨씬 큰 메모리 블록을 할당하는 리눅스 관련 기능이다.

예를 들어, 기본 page 크기가 4KiB인 시스템에서, `hugepages-2Mi: 80Mi` 제한을 설정할 수 있다. container가 40개를 초과하는 2MiB huge page(총 80BiM)를 할당하려고 하면 해당 할당은 실패한다.

**Note**: `hugepages-*` 리소스 메모리, cpu와 다르게 overcommit할 수 없다.

## Resource requests and limits of Pod and container
container 대해 아래의 `limit`, `request`를 지정할 수 있다:

- spec.containers[*].resources.limits.cpu
- spec.containers[*].resources.limits.memory
- spec.containers[*].resources.limits.hugepages-<size>
- spec.containers[*].resources.requests.cpu
- spec.containers[*].resources.requests.memory
- spec.containers[*].resources.requests.hugepages-<size>

po에 대한 `limit`, `request`는 각 container의 값을 다 더함으로써 계산하면 된다.

## Resource units in Kubernetes
### CPU resource units
k8s에 CPU에 대한 값 1은 물리 호스트 또는 가상 머신인지에 따라 **1 물리 코어** 또는 **1 가상 코어**에 해당한다.

소수점 형태로 값을 사용할 수 있다. 0.1은 100m와 같은 의미다.

CPU 리소스는 항상 리소스의 절대량으로 표시된다. 즉 코어의 개수와 상관없이 500m는 항상 같은 양의 CPU를 나타낸다.

**Note**: k8s에서 CPU 리소스를 1m보다 더 정밀한 단위로 표기할 수 없다.

### Memory resource units
메모리는 byte 단위를 사용한다. E, P, T, G, M, k, ... 또는 Ei, Pi, Ti, Gi, Mi, Ki, ...를 사용할 수 있다.

## Container resources example
아래는 2개의 container를 갖는 po에 대한 예시다.

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: frontend
spec:
  containers:
  - name: app
    image: images.my-company.example/app:v4
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
  - name: log-aggregator
    image: images.my-company.example/log-aggregator:v6
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"`
```

po는 requests 0.5 CPU, 128MiB 메모리, limits 1 CPU, 256MiB에 대한 컴퓨트 리소스를 의미한다.

## How Pods with resource requests are scheduled
po를 생성할 떄, k8s scheduler는 po를 실행할 no를 선택한다. 각 no는 po에 제공할 수 있는 최대 리소스 용량을 갖는다. scehduler는 각 리소스 유형에 대해 no에서 실행 중인 container들의 리소스 `request` 합이 no의 용량을 넘지 않도록 보장한다. 실제 no의 리소스에 대한 사용량이 낮더라도, scheduler는 `request`를 기준으로 하기 때문에 해당 no에 po를 할당하지 않는다.

## How Kubernetes applies resource requests and limits
kubelet은 container를 실행할 때, container runtime에 `request`, `limit`을 전달한다.

리눅스에서 kernel cgroups에 대해 `limit`을 적용한다.

- The CPU limit defines a hard ceiling on how much CPU time that the container can use. During each scheduling interval (time slice), the Linux kernel checks to see if this limit is exceeded; if so, the kernel waits before allowing that cgroup to resume execution.
- The CPU request typically defines a weighting. If several different containers (cgroups) want to run on a contended system, workloads with larger CPU requests are allocated more CPU time than workloads with small requests.
- The memory request is mainly used during (Kubernetes) Pod scheduling. On a node that uses cgroups v2, the container runtime might use the memory request as a hint to set memory.min and memory.low.
- The memory limit defines a memory limit for that cgroup. If the container tries to allocate more memory than this limit, the Linux kernel out-of-memory subsystem activates and, typically, intervenes by stopping one of the processes in the container that tried to allocate memory. If that process is the container's PID 1, and the container is marked as restartable, Kubernetes restarts the container.
- The memory limit for the Pod or container can also apply to pages in memory backed volumes, such as an emptyDir. The kubelet tracks tmpfs emptyDir volumes as container memory use, rather than as local ephemeral storage.

container의 메모리 request을 초과하고 해당 no의 메모리가 부족할 경우 해당 container가 속한 po가 eviction될 수 있다.

container가 비교적 오랫동안 CPU limit을 초과하는 것이 허용될 수도, 허용되지 않을 수도 있다. 하지만 container runtime은 이를 이유로 po 또는 container를 종료시키지 않는다.

### Monitoring compute & memory resource usage
kubelet은 po의 `.status`에서 po의 리소스 사용량을 노출한다.

추가적으로 클러스터에 모니터링을 위한 툴이 활성화되면 Metrics API 또는 모니터링 툴을 이용해 조회할 수 있다.

## Local ephemeral storage

## Extended resources

## PID limiting
pid에 대한 limit은 po가 사용할 수 있는 pid 개수를 제한한다. 자세한 내용은 PID Limitting 페이지를 참고한다.

## Troubleshooting