k8s에서 일부 object를 다른 object를 소유한다. 예를 들어 rs는 po를 소유할 수 있다. 이렇게 종속 object들은 소유자 object에 의존한다.

Ownership is different from the labels and selectors mechanism that some resources also use. For example, consider a Service that creates EndpointSlice objects. The Service uses labels to allow the control plane to determine which EndpointSlice objects are used for that Service. In addition to the labels, each EndpointSlice that is managed on behalf of a Service has an owner reference. Owner references help different parts of Kubernetes avoid interfering with objects they don’t control.

## Owner references in object specifications
종속 object는 소유자 object를 참조할 수 있도록 `.metadata.ownerReferences`필드가 있다. 유효한 owner reference일 경우 해당 필드는 종속 object와 동일한 ns 내의 object 이름과 UID로 구성된다. k8s는 rs, ds, deploy, jobs, cj, rc와 같이 다른 object에 종속된 object에 이 필드 값을 자동으로 설정한다. 이 필드의 값을 수정해 이러한 관계를 수동으로 설정할 수도 있다. 하지만 일반적으로 k8s가 자동으로 관리하도록 허용할 수 있다.

종속 object는 `.metadata.ownerReferences.blockOwnerDeletion` 필드에 boolean 값을 사용해 소유자 object를 삭제하지 못하도록 gc를 차단할 수 있다. k8s controller(예를 들어 deploy controller는 `.metadata.ownerReference` 필드 설정 시 이 필드를 자동으로 true로 설정한다. 물론 사용자가 직접 설정할 수도 있다.

k8s admission controller는 소유자의 삭제 권한에 따라 종속 resource에 대해 이 필드를 변경하기 위한 사용자 접근을 제어한다. 이를 통해 권한이 없는 사용자로부터 소유자 object 삭제 지연을 방지한다.

**Note**:
Cross-namespace owner references are disallowed by design. Namespaced dependents can specify cluster-scoped or namespaced owners. A namespaced owner must exist in the same namespace as the dependent. If it does not, the owner reference is treated as absent, and the dependent is subject to deletion once all owners are verified absent.

Cluster-scoped dependents can only specify cluster-scoped owners. In v1.20+, if a cluster-scoped dependent specifies a namespaced kind as an owner, it is treated as having an unresolvable owner reference, and is not able to be garbage collected.

In v1.20+, if the garbage collector detects an invalid cross-namespace ownerReference, or a cluster-scoped dependent with an ownerReference referencing a namespaced kind, a warning Event with a reason of OwnerRefInvalidNamespace and an involvedObject of the invalid dependent is reported. You can check for that kind of Event by running kubectl get events -A --field-selector=reason=OwnerRefInvalidNamespace.

## Ownership and finalizers
k8s에 resource를 삭제하도록 지시하면 API server에서 controller가 resource에 대한 finalizer 규칙을 처리할 수 있다. finalizer는 클러스터가 올바르게 동작하는 데 필요한 resource가 실수로 삭제되는 것을 방지한다. 예를 들어 po에서 아직 사용 중인 pv를 삭제하려고 하면 pv의 kubernetes.io/pv-protection finalizer가 있기 때문에 즉시 삭제되지 않는다. 대신 k8s가 finalizer가 없어질 때까지 pv는 terminating 상태로 유지되며 이는 pv가 더 이상 po에서 사용하지 않을 때 삭제된다.

k8s는 또한 foreground 또는 orphan cascading 삭제를 사용할 때 소유자 resource에 finalizer를 추가한다. foreground 삭제에서는 소유자를 삭제하기 전에 controller가 `ownerReferences.blockOwnerDeletion=true`인 종속 resource를 삭제해야 하도록 소유자 object에 foreground finalizer를 추가한다. orphan 삭제에서는 k8s가 controller가 소유자 object를 삭제한 후 종속 object를 무시하도록 orphan finalizer를 추가한다. 