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