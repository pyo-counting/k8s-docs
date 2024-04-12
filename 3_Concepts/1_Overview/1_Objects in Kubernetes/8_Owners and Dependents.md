k8s에서 일부 object는 다른 object를 소유한다. 예를 들어 rs는 po를 소유할 수 있다. 이렇게 종속 object들은 소유자 object에 의존한다.

소유에 대한 개념은 일부 resource가 사용하는 [labels and selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/)에 대한 개념과는 다르다. 예를 들어 endpointSlices를 생성하는 svc가 있다. control plane은 label을 사용해 svc에서 사용되는 endpointSlices 구분할 수 있다. label 뿐만 아니라 endpointSlices에는 owner reference가 있다. k8s의 각 controller-manager는 owner reference를 통해 자신이 제어하는 object를 구별한다.

## Owner references in object specifications
종속 object는 소유자 object를 참조할 수 있도록 `.metadata.ownerReferences`필드가 있다. 유효한 owner reference일 경우 해당 필드는 종속 object와 동일한 ns 내의 object 이름과 UID로 구성된다. k8s는 rs, ds, deploy, jobs, cj, rc와 같은 object에 종속되는 object에 이 필드 값을 자동으로 설정한다. 이 필드의 값을 수정해 이러한 관계를 수동으로 설정할 수도 있다. 하지만 일반적으로 k8s가 자동으로 관리하도록 허용할 수 있다.

아래는 rs에 종속된 po의 manifest 예시다:

``` yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: "2020-02-12T07:06:16Z"
  generateName: frontend-
  labels:
    tier: frontend
  name: frontend-b2zdv
  namespace: default
  ownerReferences:
    - apiVersion: apps/v1
      blockOwnerDeletion: true
      controller: true
      kind: ReplicaSet
      name: frontend
      uid: f391f6db-bb9b-4c09-ae74-6a1f77f3d5cf
```

종속 object는 `.metadata.ownerReferences.blockOwnerDeletion` 필드에 boolean 값을 사용해 종속 object에 대한 gc에서 소유자 object를 삭제하지 못하도록 할 수 있다. k8s controller(예를 들어 deploy)는 `.metadata.ownerReference` 필드 설정 시 이 필드를 자동으로 true로 설정한다. 물론 사용자가 직접 설정할 수도 있다.

k8s admission controller는 소유자의 삭제 권한에 따라 종속 resource에 대해 이 필드를 변경하기 위한 사용자 접근을 제어한다. 이를 통해 권한이 없는 사용자로부터 소유자 object 삭제 지연을 방지한다.

> **Note**: 
> Cross-namespace owner references are disallowed by design. Namespaced dependents can specify cluster-scoped or namespaced owners. A namespaced owner must exist in the same namespace as the dependent. If it does not, the owner reference is treated as absent, and the dependent is subject to deletion once all owners are verified absent.
>
> Cluster-scoped dependents can only specify cluster-scoped owners. In v1.20+, if a cluster-scoped dependent specifies a namespaced kind as an owner, it is treated as having an unresolvable owner reference, and is not able to be garbage collected.
> 
> In v1.20+, if the garbage collector detects an invalid cross-namespace ownerReference, or a cluster-scoped dependent with an ownerReference referencing a namespaced kind, a warning Event with a reason of OwnerRefInvalidNamespace and an involvedObject of the invalid dependent is reported. You can check for that kind of Event by running kubectl get events -A --field-selector=reason=OwnerRefInvalidNamespace.

## Ownership and finalizers
k8s에 resource를 삭제하도록 지시하면 kube-apiserver는 controller가 resource에 대한 finalizer 규칙을 처리하도록 허용한다. finalizer는 cluster가 올바르게 동작하는 데 필요한 resource가 실수로 삭제되는 것을 방지한다. 예를 들어 po에서 아직 사용 중인 pv를 삭제하려고 하면 pv의 `kubernetes.io/pv-protection` finalizer가 있기 때문에 즉시 삭제되지 않는다. 대신 k8s가 finalizer를 없앨 때까지 pv는 terminating 상태로 유지되며 이는 pv가 더 이상 po에서 사용하지 않을 때 삭제된다.

k8s는 또한 [foreground or orphan cascading deletion](https://kubernetes.io/docs/concepts/architecture/garbage-collection/#cascading-deletion)를 사용할 때 소유자 resource에 finalizer를 추가한다. foreground 삭제에서는 소유자를 삭제하기 전에 controller가 `ownerReferences.blockOwnerDeletion=true`인 종속 resource를 삭제하기 위해 소유자 object에 foreground finalizer를 추가한다. orphan 삭제에서는 k8s가 controller가 소유자 object를 삭제한 후 종속 object를 무시하도록 orphan finalizer를 추가한다. 