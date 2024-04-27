po는 k8s에서 생성 및 관리할 수 있는 가장 작은 단위의 배포 가능한 컴퓨팅 단위다.

po는 공유 storage, 네트워크 리소스 및 실행 할 container에 대한 명세를 갖는 container의 그룹이다. po의 컨텐츠는 항상 같은 공간에 있으며 shared context에서 실행된다.

po는 애플리케이션 container 뿐만 아니라 po 실행 과정에서 동작하는 init container도 포함할 수 있다. 뿐만 아니라 디버깅을 위한 ephemeral container를 포함할 수 있다.

## What is a Pod?
> **Note**:  
> po가 cluster의 각 no에서 실행될 수 있도록 container runtime을 설치해야 한다.

po의 shared context는 리눅스 namespace, cgroup, 잠재적인 격리 요소(container를 격리하는 요소와 동일)의 집합이다. po context 안에서 각 애플리에키션은 추가 격리가 적용될 수도 있다.

docker와 비교했을 때 po는 공유 namespace, 공유 filesystem volume를 갖는 container의 집합과 유사하다.

k8s cluster에서 po는 두 가지 주요 목적을 위해 사용한다.
- Pods that run a single container: "one-container-per-Pod" 모델은 k8s에서 가장 일반적으로 사용한다. 이 케이스에서는 po를 단일 containwr에 대한 wrapper라고 생각할 수 있다. k8s는 container를 직접 관리하기 보다는 po를 관리한다.
- Pods that run multiple containers that need to work together: po는 단단히 결합되어 있고 리소스를 공유해야 하는 여러 [multiple co-located containers](https://kubernetes.io/docs/concepts/workloads/pods/#how-pods-manage-multiple-containers)로 구성된 애프리케이션을 캡슐화할 수 있다. These co-located containers form a single cohesive unit.

일반적인 패턴은 아니며 container 간 결합성이 높은 경우에만 사용해야 한다.
You don't need to run multiple containers to provide replication (for resilience or capacity); if you need multiple replicas, see [Workload management](https://kubernetes.io/docs/concepts/workloads/controllers/).

## Using Pods
보통 po는 직접 생성하지 않고 workload resource를 사용한다.

### Workload resources for managing pods
보통 po는 싱글톤이더라도 직접 생성 및 관리하지 않는다. 대신 deploy, job 또는 po가 상태를 추적해야할 경우 sts를 사용해 생성 및 관리한다.

각 po는 애플리케이션의 단일 인스턴스를 실행한다. 만약 애플리케이션을 수평 확장하길 원하는 경우 여러 po를 실행해야 한다. k8s에서는 이를 replication이라고 부른다. replicated po는 보통 workload resource, controller가 그룹 단위로 생성, 관리한다.

## Working with Pods
k8s 환경에서는 싱글톤 po라도 직접 생성할 일은 거의 없다. 이는 po가 일시적이고 일회용의 목적으로 설계되었기 때문이다.

> **Note**:  
> po 내 container를 재시작하는 것과 po를 재시작하는 것과 혼동하면 안된다 po는 프로세스가 아니며 container를 구동하기 위한 환경이다. po는 삭제되기 전까지 지속된다.

po의 이름은 [DNS subdomanin](https://kubernetes.io/docs/concepts/overview/working-with-objects/names/#dns-subdomain-names) 규칙을 따라야 하지만 po hostname에 대해 예상치 않은 결과를 초래할 수 있으므로 더제 한적인 [DNS label](https://kubernetes.io/docs/concepts/overview/working-with-objects/names/#dns-label-names) 규칙을 따라야 한다.

### Pod OS
po가 실행될 OS를 지정하기 위해 po의 `.spec.os.name` 필드를 `windows` 또는 `linux`를 설정해야 한다. 현재는 두 값만 지원한다.

k8s v1.30에서 이 값은 po의 스케줄링에 영향을 미치지는 않는다. Setting the .spec.os.name helps to identify the pod OS authoritatively and is used for validation. kubelet이 실행 중인 no의 OS와 동일하지 않는 값일 경우 kubelet은 po 실행을 거부한다. 그리고 [Pod security standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)도 관련 없는 OS에 대한 policy를 제한하는 것을 피한다.

### Pods and controllers
여러 po 생성 및 관리를 위해 workload resource를 사용할 수 있다. resource의 controller는 po의 replication, roll out, 실패 상황에서의 치유를 처리한다. 예를 들어 no가 실패되면 controller가 해당 no의 po 동작이 멈춤을 인지하고 대체 po를 생성한다. 그리고 scheduler는 정상 no에 대체 po를 스케쥴링한다.

아래는 1개 이상의 po를 관리하는 workload resource 예시다.
- deploy
- sts
- ds

### Pod templates
workload resource들의 controller는 po template으로부터 po을 생성하고 관리한다.

아래는 간단한 job workload resource 예시다.
``` yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: hello
spec:
  template:
    # This is the pod template
    spec:
      containers:
      - name: hello
        image: busybox:1.28
        command: ['sh', '-c', 'echo "Hello, Kubernetes!" && sleep 3600']
      restartPolicy: OnFailure
    # The pod template ends here
```

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
> **Note**:  
> container runtime에서 privileged container 개념을 지원해야 한다.

linux, windows 환경에서 container에 privileged 옵션을 사용해 privileged mode를 사용할 수 있다.

### Linux privileged containers
linux에서 po 내 container는 [security context](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/) 내 `privileged` flag를 사용해 privileged mode를 활성화 할 수 있다. 이는 하드웨어 장치, network stack에 대한 조작과 같은 OS 관리 기능을 사용하려는 container에 유용하다.

### Windows privileged containers

## Static Pods
static po는 kube-apiserver의 관찰 없이 특정 no의 kubelet daemon에 의해 관리된다. deploy와 같이 대부분의 po는 control plane에 의해 관리되는 반면 static po는 kubelet이 직접 관리한다.

static po는 항상 특정 node의 kubelet에 한정된다. static po의 주된 용도는 자체 호스팅 control plane을 실행하는 것이다: 즉, kubelet을 사용해 개별 control plane 구성 요소를 감독한다.

kubelet은 각 static po에 대해 kube-apiserver에 mirror po(kubelet에 의해 관리되는 static po를 추적하는 object)를 자동으로 생성한다. 이는 no에 실행되는 po가 kube-apiserver에서 볼 수 있음을 의미하지만 kube-apiserver를 통해 제어는 하지 못한다. 자세한 내용은 [Create static Pods](https://kubernetes.io/docs/tasks/configure-pod-container/static-pod/)를 참고한다.

> **Note**:  
> static po의 .spec에서는 다른 API object를 참조할 수 없다(예를 들어 sa, cm, secret 등).

## Pods with multiple containers
po는 여러 container를 지원하도록 설계됐다. po 내 container는 cluster의 동일 no에 스케줄링된다. po 내 container는 리소스, 종속성을 공유하고 서로 통신할 수 있다.

Pods in a Kubernetes cluster are used in two main ways:
- Pods that run a single container. The "one-container-per-Pod" model is the most common Kubernetes use case; in this case, you can think of a Pod as a wrapper around a single container; Kubernetes manages Pods rather than managing the containers directly.
- Pods that run multiple containers that need to work together. A Pod can encapsulate an application composed of multiple co-located containers that are tightly coupled and need to share resources. These co-located containers form a single cohesive unit of service—for example, one container serving data stored in a shared volume to the public, while a separate sidecar container refreshes or updates those files. The Pod wraps these containers, storage resources, and an ephemeral network identity together as a single unit.

예를 들어 아래와 같이 po 내 shared volume의 파일을 통해 웹 서버를 제공하는 container와 원격 소스로부터 파일을 업데이트하는 [sidecar container](https://kubernetes.io/docs/concepts/workloads/pods/sidecar-containers/)를 가질 수 있다.

![](https://kubernetes.io/images/docs/pod.svg)

Some Pods have init containers as well as app containers. By default, init containers run and complete before the app containers are started.

You can also have sidecar containers that provide auxiliary services to the main application Pod (for example: a service mesh).

Enabled by default, the SidecarContainers feature gate allows you to specify restartPolicy: Always for init containers. Setting the Always restart policy ensures that the containers where you set it are treated as sidecars that are kept running during the entire lifetime of the Pod. Containers that you explicitly define as sidecar containers start up before the main application Pod and remain running until the Pod is shut down.

## Container probes
probe는 kubelet에 의해 주기적으로 container를 대상으로 수행된다. kubelet은 다른 유형의 probe를 수행할 수 있다.
- ExecAction (container runtime의 도움으로 수행)
- TCPSocketAction (kubelet에 의해 직접 수행)
- HTTPGetAction (kubelet에 의해 직접 수행)