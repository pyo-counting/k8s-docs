### How a ReplicaSet works
- rs은 po의 .metadata.ownerReferences를 통해 po에 연결되며 이는 현재 어떤 resource 소유인지를 구분한다. rs는 이 연결 정보를 통해 po를 관리한다. rs의 label matcher와 일치하면서 OwnerReference가 없는 po(controller 예외)는 해당 rs에 소유된다.

### When to use a ReplicaSet
- rs는 항상 선언된 replica 수 만큼 po를 유지하는 것을 보장한다. 하지만 더 높은 수준의 개념인 deploy는 rs를 관리하고 po의 declarative update 및 여러 기능을 제공한다. 그렇기 때문에 직접 rs를 사용하기 보다는 deploy 사용을 권장한다.

### Non-Template Pod acquisitions
- 직접 po를 생성하는 것에는 제약이 없지만 rs의 selector와 매치되는 label을 갖지 않도록 주의해야 한다. 왜냐하면 rs는 .spec.template으로 생성된 po만 소유할 수 있도록 제약이 있지 않기 때문이다.

  - rs => po 생성 시, 새로 생성된 po는 rs의 replicas 개수를 초과하기 때문에 바로 종료된다.
  - po => rs 생성 시, rs의 template으로 생성되는 po는 replicas 개수보다 더 적게 생성된다.

### Pod Template
- 다른 controller가 po를 관리하지 않도록 하기 위해 selector가 겹치지 않도록 주의해야 한다. po의 restart policy 필드(.spec.template.spec.restartPolicy)는 기본값인 Always만 허용된다.

### Pod Selector
- .spec.template.metadata.labels는 .spec.selector와 매치되어야하며 그렇지 않을 경우 API에 의해 거절된다.

### Replicas
- .spec.replicas를 명시하지 않으면 기본 값 1을 갖는다.

### Deleting a ReplicaSet and its Pods
kubectl delete 명령어를 사용해 rs와 po을 삭제할 수 있다. 기본적으로 garbage collector가 의존 po를 모두 삭제한다.

### Deleting just a ReplicaSet
--cascade=orphan 옵션을 사용해 rs만 삭제할 수도 있다.

기존 rs가 삭제되고 동일한 .spec.selector울 갖는 새로운 rs가 생성된다면 기존 po를 소유하게된다. 하지만 기존 po를 새로운 template과 일치시키 위한 동작은 하지 않는다. 새로운 template의 po로 업데이트 되는 기능을 사용하고자 할 경우 deploy를 대신 사용해야 한다. rs는 직접 rolling update를 지원하지 않는다.

### Isolating Pods from a ReplicaSet
po의 label를 변경해 rs의 관리하에서 벗어날 수 있다. 이러한 기능은 디버깅과 같은 작업 시 유용하다. 물론 이렇게 제거된 po를 대체할 po가 다시 생성된다.

### Scaling a ReplicaSet
.spec.replicas 값을 조절해 scale up, down을 수행할 수 있다. rs는 label selector와 매칭되는 po의 수를 보장한다.

scale down시 rs는 아래와 같은 일반적인 알고리즘을 사용해 삭제할 po를 정렬 및 선택한다:

1. pending(또는 unschedulable) po를 먼저 scale down한다.
2. controller.kubernetes.io/pod-deletion-cost annotation이 있을 경우 값이 낮은 po가 우선순위가 높다
3. no에 po 개수가 많은 경우 우선 순위가 높다.
4. po 생성 시간이 다를 경우 가장 최근에 생성된 po의 우선 순위가 더 높다(LogarithmicScaleDown 기능이 활성화 된 경우 생성 시간은 log 형태 정수로 저장된다).

위 모든 사항에 대해 매칭되는 경우 선택은 랜덤이다.

### Pod deletion cost
controller.kubernetes.io/pod-deletion-cost annotation을 사용해 사용자는 rs를 scale down할 때의 삭제할 po의 우선순위를 조절할 수 있다.

값의 범위는 [-2147483647, 2147483647]이며 po에 annotation으로 추가해야 한다. 이는 동일한 rs에 속한 po간의 삭제에 대한 비용을 나타낸다. 삭제 비용이 낮은 po는 다른 높은 삭제 비용을 갖는 po보다 삭제에 대한 우선순위가 높다.

해당 annotation을 지정하지 않은 po의 암묵적인 값은 0이다. 위 값의 범위를 벗어날 경우 API server에 의해 거절된다.

이 기능은 베타로 기본적으로 활성화된다. 물론 kube-apiserver, kube-controller-manager에서 feature gate PodDeletionCost를 사용해 비활성화 할 수 있다.

- 이는 po 삭제 순서에 대한 보장을 하지는 않는다.
- metric 값을 토대로 annotation을 자주 업데이트하는 것을 피해야 한다. 이는 API server에 상당한 수의 po 업데이트를 생성하기 때문이다.

### Example Use Case
특정 애플리케이션에 대한 여러 po는 각각이 다른 utilization level을 가질 수 있다. 그렇기 때문에 scale down시 낮은 utilization의 po를 제거하는 것을 선호할 수 있다. po를 자주 업데이트 하지 않기 위해 scale down 전에 controller.kubernetes.io/pod-deletion-cost(po utilization level에 대한 비율)를 한 번 업데이트해야 한다. 이는 애플리케이션 자체가 scale down을 제어할 경우 동작한다(예를 들어 Spark 배포의 드라이버 po).

### ReplicaSet as a Horizontal Pod Autoscaler Target
rs는 HPA(Horizontal Pod Autoscalers)의 target이 될 수 있다. 즉 rs는 HPA에 의해 auto-scale될 수 있다.