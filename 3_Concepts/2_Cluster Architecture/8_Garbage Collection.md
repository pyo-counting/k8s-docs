gc는 k8s 클러스터가 resource를 정리하기 위해 사용하는 다양한 방법을 종합하는 용어다. 다음과 같은 resource를 정리한다:

- [Terminated pods](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-garbage-collection)
- [Completed Jobs](https://kubernetes.io/docs/concepts/workloads/controllers/ttlafterfinished/)
- Objects without owner references
- Unused containers and container images
- [Dynamically provisioned PersistentVolumes with a StorageClass reclaim policy of Delete](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#delete)
- [Stale or expired CertificateSigningRequests (CSRs)](https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests/#request-signing-process)
- Nodes deleted in the following scenarios:
    - On a cloud when the cluster uses a [cloud controller manager](https://kubernetes.io/docs/concepts/architecture/cloud-controller/)
    - On-premises when the cluster uses an addon similar to a cloud controller manager
- [Node Lease objects](https://kubernetes.io/docs/concepts/architecture/nodes/#heartbeats)

## Owners and dependents
k8s의 많은 object들은 owner referenece를 통해 서로 연결된다. control plane은 owner reference 정보를 통해 object들의 소유 관계를 확인한다. object를 삭제하기 전에 control plane, API client는 owner reference를 사용해 관련 리소스를 정리할 수 있다. 대부분의 경우 k8s가 자동으로 owner reference를 관리한다.

owner reference는 일부 리소스에서 사용하는 label selector 메커니즘과 다르다. 예를 들어 EndpointSlice를 생성하는 svc의 경우, svc는 label을 사용해 control plane이 해당 svc에서 사용하는 EndpointSlice object를 결정할 수 있도록 도와준다. 추가적으로 EndpointSlice에는 owner reference 정보도 존재한다. Owner references help different parts of Kubernetes avoid interfering with objects they don’t control.

> **Note**:  
> Cross-namespace owner references are disallowed by design. Namespaced dependents can specify cluster-scoped or namespaced owners. A namespaced owner must exist in the same namespace as the dependent. If it does not, the owner reference is treated as absent, and the dependent is subject to deletion once all owners are verified absent.
>
> Cluster-scoped dependents can only specify cluster-scoped owners. In v1.20+, if a cluster-scoped dependent specifies a namespaced kind as an owner, it is treated as having an unresolvable owner reference, and is not able to be garbage collected.
>
> In v1.20+, if the garbage collector detects an invalid cross-namespace ownerReference, or a cluster-scoped dependent with an ownerReference referencing a namespaced kind, a warning Event with a reason of OwnerRefInvalidNamespace and an involvedObject of the invalid dependent is reported. You can check for that kind of Event by running kubectl get events -A --field-selector=reason=OwnerRefInvalidNamespace.

## Cascading deletion
k8s는 object를 삭제할 때 더 이상 owner reference가 없는지 확인한다. 예를 들어 rs을 삭제할 때 남겨진 po가 없는지 확인하고 삭제한다. k8s가 object를 삭제할 때 cascading deletion 프로세스를 사용해 종속 object를 자동으로 삭제할지 여부를 제어할 수 있다. cascading deletion에는 다음과 같은 두 가지 종류가 있다:

- Foreground cascading deletion
- Background cascading deletion

또한 k8s의 finalizers를 사용해 gc가 owner reference가 있는 resource을 언제 어떻게 삭제할지 제어할 수 있다.

### Foreground cascading deletion
foreground cascading deletion에서는 삭제하려는 소유자 object가 먼저 삭제 중 상태가 된다. 이 상태에서는 소유자 object에 다음과 같은 일이 일어난다:
- kube-apiserver가 object의 `.metadata.deletionTimestamp` 필드를 object가 삭제 마킹된 시간으로 설정한다.
- kube-apiserver가 `.metadata.finalizers` 필드를 foregroundDeletion로 설정한다.
- object는 삭제 과정이 완료되기 전까지 kube-apiserver를 통해 조회할 수 있다.

소유자 object가 삭제 중 상태가 된 이후 controller는 종속 object를 삭제한다. 모든 종속 object가 삭제되면 controller가 소유자 object를 삭제한다. 이 시점에서 object는 더 이상 kube-apiserver를 통해 조회할 수 없다.

foreground cascading deletion 중에 소유자 object의 삭제를 막는 종속 object는 `ownerReference.blockOwnerDeletion` 필드 값이 true인 object다. 더 자세한 내용은 [Use foreground cascading deletion](https://kubernetes.io/docs/tasks/administer-cluster/use-cascading-deletion/#use-foreground-cascading-deletion)를 참고한다.

### Background cascading deletion
background cascading deletion에서는 kube-apiserver가 소유자 object를 즉시 삭제하고 백그라운드에서 controller가 종속 object들을 삭제한다. k8s는 기본적으로 background cascading deletion를 사용한다.

### Orphaned dependents
k8s가 소유자 object를 삭제할 때 삭제되지 않고 남은 종속 object를 orphan object라고 부른다. 기본적으로 k8s는 종속 object를 삭제한다. 하지만 orphan cascade를 사용해 삭제되지 않도록 할 수 있다.

## Garbage collection of unused containers and images
kubelet은 사용되지 않는 image에 대한 gc를 2분, container에 대한 gc를 1분마다 수행한다. 외부 gc 도구는 kubelet의 행동을 방해하고 필요한 container를 삭제할 수 있으므로 사용을 피해야 한다.

사용되지 않는 container와 image에 대한 gc 옵션을 설정하기 위해 configuration file 사용하여 kubelet을 수정하고 KubeletConfiguration 리소스 타입의 gc과 관련된 파라미터를 수정한다.

### Container image lifecycle
k8s는 kubelet의 일부인 image manager가 cadvisor와 협동하여 모든 image의 라이프사이클을 관리한다. kubelet은 gc 결정을 내릴 때 다음 디스크 사용량 제한을 고려한다:

- HighThresholdPercent
- LowThresholdPercent

HighThresholdPercent 값을 초과한 디스크 사용량은 마지막으로 사용된 시간을 기준으로 오래된 image 순서대로 삭제하는 gc를 트리거한다. kubelet은 디스크 사용량이 LowThresholdPercent 값에 도달할 때까지 image를 삭제한다.

## Garbage collection for unused container images
alpha 기능으로 디스크 사용량과 무관하게 로컬에 있는 사용되지 않는 image의 최대 시간을 설정할 수 있다. 이는 각 no의 kubelet에 대한 설정이다.

이 기능을 사용하기 위해 kubelet의 ImageMaximumGCAge feature gate를 활성화하고 kubelet 설정 파일에서 ImageMaximumGCAge 필드를 사용하면 된다.

값은 k8s에서 사용하는 duration 형태를 사용한다. 예를 들어 3일 12시간은 3d12h로 표현한다.

### Container garbage collection
kubelet은 사용자가 정의할 수 있는 다음 변수들을 기반으로 사용되지 않는 container를 gc한다:
- MinAge: kubelet이 gc할 수 있는 container의 최소 나이. 0으로 설정해 비활성화할 수 있다.
- MaxPerPodContainer: 각 po가 가질 수 있는 죽은 container의 최대 개수. 0으로 설정해 비활성화할 수 있다.
- MaxContainers: 클러스터가 가질 수 있는 죽은 container의 최대 개수. 0으로 설정해 비활성화할 수 있다.

위 변수와 더불어 kubelet은 식별할 수 없고 삭제된 container를 오래된 순서부터 gc한다.

MaxPerPodContainer와 MaxContainers는 po의 최대 container 개수(MaxPerPodContainer)를 유지하는 것이 전체 죽은 container의 개수 제한(MaxContainers)을 초과하게 될 때, 서로 충돌이 발생할 수 있다. 이 상황에서 kubelet은 충돌을 해결하기 위해 MaxPerPodContainer를 조절한다. 최악의 시나리오에서는 MaxPerPodContainer를 1로 다운그레이드하고 가장 오래된 컨테이너들을 eviction한다. 또한, 삭제된 po가 소유한 container들은 MinAge보다 오래되었을 때 삭제된다.

> **Note**:  
> kubelet은 자신이 관리하는 container에 대한 gc만 수행한다.

## Configuring garbage collection
resource를 관리하는 controller의 옵션을 설정해 gc를 수정할 수 있다. 아래 페이지에서는 어떻게 gc를 설정할 수 있는지 확인할 수 있다:
- [Configuring cascading deletion of Kubernetes objects](https://kubernetes.io/docs/tasks/administer-cluster/use-cascading-deletion/)
- [Configuring cleanup of finished Jobs](https://kubernetes.io/docs/tasks/administer-cluster/kubelet-config-file/)