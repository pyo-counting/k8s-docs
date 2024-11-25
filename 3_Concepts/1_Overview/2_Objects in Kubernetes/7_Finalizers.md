finalizer는 삭제 마킹된 resource를 완전히 삭제하기 전에 특정 조건이 충족될 때까지 대기하도록 k8s에 지시하는 namespaced key다. finalizer는 controller로 하여금 삭제된 object가 소유한 resource를 정리하도록 한다.

finalizer가 존재하는 object를 삭제하도록 k8s에 지시하면 k8s API는 `.metadata.deletionTimestamp`를 추가해 해당 object를 삭제 대상으로 마킹하고 HTTP 202 status code를 반환한다. control plane 또는 다른 구성 요소가 finalizer가 정의한 작업을 수행하는 동안 대상 object는 terminating 상태를 유지한다. 작업이 완료되면 controller는 대상 object에서 finalizer를 삭제한다. `.metadata.finalizers`가 빈 값이면 k8s는 삭제가 완료된 것으로 간주하고 해당 object를 삭제한다.

finalizer를 사용해 resource에 대한 gc를 제어할 수 있다. 예를 들어 controller가 대상 resource를 삭제하기 전에 관련 resource 또는 인프라를 정리하도록 finalizer를 정의할 수 있다.

finalizer를 사용해 대상 resoucre를 삭제하기 전에 특정 정리 작업을 수행하도록 controller에게 알림으로써 resource의 gc를 제어할 수 있다.

finalizer는 일반적으로 실행항 코드를 지정하지 않는다. 대신 일반적으로 annotation과 유사한 특정 resource에 대한 key 목록이다. k8s는 일부 finalizer를 자동으로 지정하지만 사용자가 직접 설정할 수도 있다. 아래는 finalizer를 갖는 pv의 메니페스트 파일 일부 내용이다.
``` yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  finalizers:
    - kubernetes.io/pv-protection
```

## How finalizers work
manifest 파일을 사용해 resource를 생성할 떄 `.metadata.finalizers` 필드에 finalizer를 명시할 수 있다. 해당 resource를 삭제하려고 할 때 kube-apiserver는 finalizer 필드 값을 확인하고 다음을 수행한다.
- 삭제 요청 시간을 `.metadata.deletionTimestamp` 필드 값을 추가해 ojbect를 수정한다.
- `metadata.finalizers` 필드가 빈 상태가 될때까지 object가 삭제되지 않도록 한다.
- HTTP 202 status code(Accepted)를 반환한다.

finalizer를 관리하는 controller는 object 삭제가 요청됐음을 나타내는 `.metadata.deletionTimestamp` 필드 설정에 대한 업데이트를 확인한다. controller는 해당 resource에 대해 finalizer의 요구 사항을 충족하려고한다. finalizer 조건이 만족될 때마다 controller는 resource의 finalizer 필드에서 해당 키를 삭제한다. finalizer 필드가 빈 값이 되면 deletionTimestamp가 설정된 object가 자동으로 삭제된다. finalizer를 사용해 관리되지 않는 resource의 삭제를 방지할 수 있다.

finalizer에 대한 일반적인 사용 예는 kubernetes.io/pv-protection이 있으며 이는 pv 객체의 우발적인 삭제를 방지하기 위함이다. pv 객체가 po에서 사용 중인 경우 k8s는 pv에 finalizer를 추가한다. 해당 pv를 삭제하려고 하면 terminating 상태가 되지만 finalizer가 존재하기 때문에 controller에서 해당 object를 삭제할 수 없다. po가 pv 사용을 중지하면 k8s는 pv-protection finalizer를 삭제하고 controller는 volume을 삭제한다.
> **Note**:  
> - object를 삭제하면 k8s는 deletion timestamp를 추가하고 삭제 pending 상태의 object의 `.metadata.finalizers` 필드에 대한 변경을 제한한다.
> - 삭제가 요청된 이후 object를 다시 부활시킬 수 없다. 유일한 방법은 object를 삭제하고 유사한 object를 다시 생성하는 것이다.

## Owner references, labels, and finalizers
owner reference는 label와 마찬가지로 k8s 내 resource object 간 관계를 설명하지만 다른 목적으로 사용된다. controller가 po와 같은 object를 관리할 때 label을 사용해 관련 object 그룹의 변경 사항을 추적한다. 예를 들어 job이 po를 생성할 때 job controller는 해당 po에 label을 적용하고 동일한 label이 있는 클러스터의 모든 po에 대해 변경 사항을 추적한다.

job controller는 뿐만 아니라 po에 owner reference를 추가해 po를 생성한 job를 참조한다. 이러한 po가 실행되는 동한 job을 삭제하면 k8s는 owner reference(label이 아님)를 사용해 클러스터에서 정리가 필요한 po를 결정한다.

또한 k8s는 삭제 대상 resource에 대한 owner reference를 식별할 때 finalizer도 처리한다.

일부 상황에서 finalizer는 종속 object의 삭제를 차단할 수 있으며, 이로 인해 소유자 object가 완전히 삭제되지 않고 예상보다 오래 유지될 수 있다. 이러한 상황에서 소유자, 종속 object에 대한 finalizer, owner reference를 확인해 원인을 해결해야 한다.

> **Note**:  
> object가 삭제 상태에 있는 경우 삭제를 계속 진행할 수 있도록 finalizer를 수동으로 삭제하면 안된다. finalizer는 일반적으로 어떤 이유로 resource에 추가되므로 강제로 삭제하면 클러스터에 문제가 발생할 수도 있다. 이는 finalizer의 목적을 이해하고 다른 방식으로 처리하기 원할 때만 수행해야 한다(예를 들어 종속 object를 수동으로 정리).