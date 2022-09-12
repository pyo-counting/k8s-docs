gc는 k8s 클러스터가 resource를 정리하기 위해 사용하는 다양한 방법을 종합한 용어다. 다음과 같은 resource를 정리한다:

- Terminated pods
- Completed Jobs
- Objects without owner references
- Unused containers and container images
- Dynamically provisioned PersistentVolumes with a StorageClass reclaim policy of Delete
- Stale or expired CertificateSigningRequests (CSRs)
- Nodes deleted in the following scenarios:
    - On a cloud when the cluster uses a cloud controller manager
    - On-premises when the cluster uses an addon similar to a cloud controller manager
- Node Lease objects

## Owners and dependents

## Cascading deletion
k8s는 object를 삭제할 때 더 이상 owner reference가 없는지 확인한다. 예를 들어 rs을 삭제할 때 남겨진 po가 없는지 확인하고 삭제한다. object를 삭제할 때 k8s가 object의 종속 object를 자동으로 삭제할 지 여부를 제어할 수 있다. 이 과정을 cascading 삭제라고 한다. cascading 삭제에는 다음과 같은 두 가지 종류가 있다:

- Foreground cascading deletion
- Background cascading deletion

또한 k8s의 finalizers를 사용해 gc가 owner reference가 있는 resource을 언제 어떻게 삭제할지 제어할 수 있다.

### Foreground cascading deletion
foreground cascading deletion에서는 삭제하려는 소유자 object가 먼저 삭제 중 상태가 된다. 이 상태에서는 소유자 object에 다음과 같은 일이 일어난다:

- k8s API server가 object의 `.metadata.deletionTimestamp` 필드를 object가 삭제 마킹된 시간으로 설정한다.
- k8s API server가 `.metadata.finalizers` 필드를 foregroundDeletion로 설정한다.
- object는 삭제 과정이 완료되기 전까지 k8s API를 통해 조회할 수 있다.
- 소유자 object가 삭제 중 상태가 된 이후, controller는 종속 object를 삭제한다. 모든 종속 object가 삭제되면 controller가 소유자 object를 삭제한다. 이 시점에서 object는 더 이상 k8s API를 통해 조회할 수 없다.

foreground cascading deletion 중에 소유자 object의 삭제를 막는 종속 object는 `ownerReference.blockOwnerDeletion` 필드 값이 true인 object다. 더 자세한 내용은 [Use foreground cascading deletion](https://kubernetes.io/docs/tasks/administer-cluster/use-cascading-deletion/#use-foreground-cascading-deletion)를 참고한다.

### Background cascading deletion
background cascading deletion에서는 k8s API server가 소유자 object를 즉시 삭제하고 백그라운드에서 controller가 종속 object들을 삭제한다. k8s는 기본적으로 background cascading deletion를 사용한다.

### Orphaned dependents
k8s가 소유자 object를 삭제할 때 삭제되지 않고 남은 종속 object를 orphan object라고 부른다. 기본적으로 k8s는 종속 object를 삭제한다.

## Garbage collection of unused containers and images
kubelet은 사용되지 않는 image에 대한 gc를 5분, container에 대한 gc를 1분마다 수행한다. 외부 gc 도구는 kubelet의 행동을 중단시키고 필요한 container를 삭제할 수 있으므로 사용을 피해야 한다.

사용되지 않는 container와 image에 대한 gc 옵션 설정은 configuration file 사용하여 kubelet 을 수정하거나 KubeletConfiguration resource 타입의 gc과 관련된 파라미터를 수정한다.

### Container image lifecycle
k8s는 kubelet의 일부인 image manager가 cadvisor와 협동하여 모든 image의 라이프사이클을 관리한다. kubelet은 gc 결정을 내릴 때 다음 디스크 사용량 제한을 고려한다:

- HighThresholdPercent
- LowThresholdPercent

HighThresholdPercent 값을 초과한 디스크 사용량은 마지막으로 사용된 시간을 기준으로 오래된 image 순서대로 삭제하는 gc를 트리거한다. kubelet은 디스크 사용량이 LowThresholdPercent 값에 도달할 때까지 이미지를 삭제한다.

### Container garbage collection
kubelet은 사용자가 정의할 수 있는 다음 변수들을 기반으로 사용되지 않는 container를 삭제한다:

- MinAge: kubelet이 gc할 수 있는 최소 나이. 0으로 설정해 비활성화할 수 있다.
- MaxPerPodContainer: 각 po가 가질 수 있는 죽은 container의 최대 개수. 0으로 설정해 비활성화할 수 있다.
- MaxContainers: 클러스터가 가질 수 있는 죽은 container의 최대 개수. 0으로 설정해 비활성화할 수 있다.

위 변수와 더불어 kubelet은 식별할 수 없고 삭제된 container 오래된 순서대로 gc한다.

MaxPerPodContainer와 MaxContainer는 po의 최대 container 개수(MaxPerPodContainer)를 유지하는 것이 전체 죽은 container의 개수 제한(MaxContainers)을 초과하게 될 때, 서로 충돌이 발생할 수 있다. 이 상황에서 kubelet은 충돌을 해결하기 위해 MaxPodPerContainer를 조절한다. 최악의 시나리오에서는 MaxPerPodContainer를 1로 다운그레이드하고 가장 오래된 컨테이너들을 축출한다. 또한, 삭제된 po가 소유한 container들은 MinAge보다 오래되었을 때 삭제된다.

**Note**: kubelet은 자신이 관리하는 container에 대한 gc만 수행한다.

## Configuring garbage collection
resource를 관리하는 controller의 옵션을 설정해 gc를 수정할 수 있다. 아래 페이지에서는 어떻게 gc를 설정할 수 있는지 확인할 수 있다:

- [Configuring cascading deletion of Kubernetes objects](https://kubernetes.io/docs/tasks/administer-cluster/use-cascading-deletion/)
- [Configuring cleanup of finished Jobs](https://kubernetes.io/docs/tasks/administer-cluster/kubelet-config-file/)