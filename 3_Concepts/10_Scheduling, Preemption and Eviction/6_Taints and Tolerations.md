no affinity는 no 집합(요구사항 또는 선호도)을 필터링 및 선택할 수 있는 po의 속성이다. 반면에 taint는 그 반대로 no가 po 집합을 제외할 수 있다.

toleration은 po에 적용된다. toleration을 통해 scheduler는 매칭되는 taint가 있는 no에 po를 스케줄할 수 있다. toleration은 스케줄링을 허용하지만 보장하지는 않는다: scheduler는 스케줄링을 위한 다른 파라미터도 평가하기 때문이다.

taint, toleration은 함께 동작하여 po가 부적절한 no에 스케줄되지 않게 한다. 하나 이상의 taint가 no에 적용된다. 이것은 no가 taint를 용인하지 않는 po를 수용해서는 안 되는 것을 나타낸다.

## Concepts
아래와 같이 `kubectl taint` 명령어를 사용해 no에 taint를 추가할 수 있다:
``` sh
kubectl taint nodes node1 key1=value1:NoSchedule
```

node1 no에 taint를 추가한다. taint에는 key=key1, vale=value1, taint effect=NoSchedule 이 있다. 이는 일치하는 toleration이 없으면 po를 node1 에 스케줄링할 수 없음을 의미한다.

아래 명령어를 사용해 taint를 삭제할 수 있다.
``` sh
kubectl taint nodes node1 key1=value1:NoSchedule-
```

po의 `.spec.tolerations` 필드에 toleration을 명시할 수 있다. 아래 toleration은 위에서 명시한 taint에 매칭되기 때문에, 아래 두 toleration 중 하나를 사용하는 po는 node1에 스케줄링될 수 있다.
``` yaml
tolerations:
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoSchedule"
```
``` yaml
tolerations:
- key: "key1"
  operator: "Exists"
  effect: "NoSchedule"
```

기본 kube-scheduler는 특정 po를 실행할 no를 선택할 때 taint와 toleration을 고려한다. 그러나 po에 `.spec.nodeName`를 명시하면 scheduler의 동작과 관련 없이 po는 해당 no에 binding된다(no에 `NoSchedule` taint가 있더라도). 물론 no에 `NoExecute` taint가 추가적으로 있는 경우에 kubelet이 po가 적절한 toleration이 없다면 실행하지 않는다.

아래는 toleration을 사용하는 po 예시다.
``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  tolerations:
  - key: "example-key"
    operator: "Exists"
    effect: "NoSchedule"
```

operator의 기본 값은 Equal이다. toleration의 key, effect가 taint와 동일하고 아래 조건을 만족하면 매칭된다.
- operator가 Exists(이 경우 value를 명시하지 않음)
- operator가 Equal이고 value가 동일

> **Note**:  
> 다음 두 가지 특별한 경우가 있다.
> - 빈 key와 operator가 Exists일 경우에 모든 key에 매칭된다. 물론 effect는 같아야한다.
> - 빈 effect는 key가 key1인 모든 effect에 매칭된다.

위의 예는 `NoSchedule` effect 를 사용했다. `PreferNoSchedule` effect를 사용할 수도 있다.

effect 필드에 사용할 수 있는 값은 다음과 같다.
- `NoExecute`: no에서 이미 실행 중인 아래 po에 영향을 미친다.
  - 매칭되는 toleration이 없는 po는 즉시 eviction된다.
  - 매칭되는 toleration에 대해 tolerationSeconds가 없는 po는 해당 no에 계속 bound된 상태로 남는다.
  - 매칭되는 toleration에 대해 tolerationSeconds가 있는 po는 해당 시간 동안 bound된 상태로 남아있으며 시간이 초과하면 eviction된다.
- `NoSchedule`: 스케줄링 단계의 po에 영향을 미친다. 매칭되는 toleration이 없는 po는 해당 no에 스케줄링 될 수 없다. 미이 실행 중인 po는 eviction되지 않는다.
- `PreferNoSchedule`: `NoSchedule`의 soft 버전이다. 시스템은 no의 taint를 허용하지 않는 po를 스케줄링하지 않으려고 노력하지만 반드시는 아니다.

동일한 no에 여러 taint를, 동일한 po에 여러 toleration을 설정할 수 있다. k8s가 여러 taint, toleration을 처리하는 방식은 필터와 같다: no의 모든 taint와 po의 모든 toleration을 비교하고 매칭되는 taint는 제외한다. 그리고 제외되지 않은 나머지 taint에 대해 po에 effect가 적용된다.
- NoSchedule effect가 있는 무시되지 않은 taint가 하나 이상 있으면 k8s는 해당 no에 po를 스케줄하지 않는다.
- NoSchedule effect가 있는 무시되지 않은 taint가 없지만 PreferNoSchedule effect가 있는 무시되지 않은 taint가 하나 이상 있으면 k8s는 po를 해당 no에 스케쥴하지 않으려고 시도한다.
- NoExecute effect가 있는 무시되지 않은 taint가 하나 이상 있으면 po가 해당 no에서 eviction되고(no에서 이미 실행 중인 경우), no에서 스케줄되지 않는다(아직 실행되지 않은 경우).

예를 들어 아래와 같은 no가 있을 경우
``` sh
kubectl taint nodes node1 key1=value1:NoSchedule
kubectl taint nodes node1 key1=value1:NoExecute
kubectl taint nodes node1 key2=value2:NoSchedule
```

아래와 같은 po가 있을 경우
``` yaml
tolerations:
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoSchedule"
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoExecute"
```

이 경우, 세 번째 taint와 일치하는 toleration이 없기 때문에 po는 no에 스케줄 될 수 없다. 그러나 no에서 이미 해당 po가 실행 중인 경우 po는 계속 실행할 수 있다.

일반적으로, NoExecute effect가 있는 taint가 no에 추가되면 taint를 용인하지 않는 po는 즉시 축출되고, taint를 용인하는 po는 축출되지 않는다. 그러나 NoExecute effect가 있는 toleration은 tolerationSeconds 옵션 필드를 설정함으로써 taint가 추가된 후 po가 no에 binding 시간을 지정하는 수 있다. 아래는 예시다.
``` yaml
tolerations:
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoExecute"
  tolerationSeconds: 3600
```

이는 po가 실행 중이고 일치하는 taint가 no에 추가되면 po는 3600초 동안 no에 binding된 후 eviction된다는 것을 의미한다. 그 전에 taint를 제거하면 po가 eviction되지 않는다.

## Example Use Cases
taint, toleration은 po를 no에서 멀어지게 하거나 실행되지 않아야 하는 po를 축출할 수 있는 유연한 방법이다. 사용 케이스는 다음과 같다.
- 전용 no: 특정 사용자들이 독점적으로 사용하도록 no 집합을 설정할 경우, 해당 no에 taint를 추가(예: `kubectl taint nodes nodename dedicated=groupName:NoSchedule`)한 다음 해당 toleration을 일부 po에 추가할 수 있다(사용자 정의 [admission controller](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/)를 작성하면 가장 쉽게 수행할 수 있음). 그런 다음 toleration이 있는 po는 taint된(전용) no와 cluster의 다른 no를 사용할 수 있다. 만약 추가적으로 다른 no를 제외한 해당 no만 사용하도록 하고 싶다면 taint와 유사한 label을 추가해야 하고(예: dedicated=groupName), admission controller는 추가로 po가 dedicated=groupName 으로 label이 지정된 no에만 스케줄될 수 있도록 no affinity를 추가해야 한다.
- 특별한 하드웨어가 있는 no: 작은 no 집합에 특별한 하드웨어(예: GPU)가 있는 cluster에서는, 특별한 하드웨어가 필요하지 않는 po를 해당 no에서 분리하여, 나중에 도착하는 특별한 하드웨어가 필요한 po를 위한 공간을 남겨두는 것이 바람직하다. 이는 특별한 하드웨어가 있는 no(`kubectl taint nodes nodename special=true:NoSchedule` 또는 `kubectl taint nodes nodename special=true:PreferNoSchedule`)에 taint를 추가하고 특별한 하드웨어를 사용하는 po에 해당 toleration을 추가하여 수행할 수 있다. 전용 no 사용 사례와 같이, 사용자 정의 [admission controller](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/)를 사용하여 toleration를 적용하는 것이 가장 쉬운 방법이다. 예를 들어, [Extended Resources](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#extended-resources)를 사용하여 특별한 하드웨어를 나타내고, 확장된 리소스 이름으로 특별한 하드웨어 no를 taint시키고 [ExtendedResourceToleration](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#extendedresourcetoleration) admission controller를 실행하는 것을 권장한다. 이제 no가 taint되었으므로, toleration이 없는 po는 스케줄되지 않는다. 그러나 확장된 리소스를 요청하는 po를 제출하면, ExtendedResourceToleration admission controller가 po에 올바른 toleration을 자동으로 추가하고 해당 po는 특별한 하드웨어 no에서 스케줄된다. 이렇게 하면 이러한 특별한 하드웨어 no가 해당 하드웨어를 요청하는 po가 전용으로 사용하며 po에 toleration을 수동으로 추가할 필요가 없다.
- taint 기반 eviction: no 문제가 있을 때 po 별로 구성 가능한 eviction 동작은 다음 섹션에서 설명한다.

## Taint based Evictions
node controller는 특정 조건이 만족될 때 pod eviction을 위해 자동으로 no를 taint시킨다. 다음은 내장 taint 목록이다.
- `node.kubernetes.io/not-ready`: no가 준비되지 않았다. 이는 NodeCondition Ready 가 "False"로 됨에 해당한다.
- `node.kubernetes.io/unreachable`: no가 no controller에서 도달할 수 없다. 이는 NodeCondition Ready 가 "Unknown"로 됨에 해당한다.
- `node.kubernetes.io/memory-pressure`: no에 memory pressure이 있다.
- `node.kubernetes.io/disk-pressure`: no에 disk pressure이 있다.
- `node.kubernetes.io/pid-pressure`: no에 PID pressure이 있다.
- `node.kubernetes.io/network-unavailable`: no의 네트워크를 사용할 수 없다.
- `node.kubernetes.io/unschedulable`: no를 스케줄할 수 없다(예를 들어 no의 `.spec.unschedulable` 필드가 false일 때).
- `node.cloudprovider.kubernetes.io/uninitialized`: kubelet의 "external" cloud provider와 같이 실행되는 경우 사용 불가능한 no로 표기하기 위해 taint를 추가한다. 이후, cloud-controller-manager의 controller가 이 no를 초기화하면 kubelet은 taint를 제거한다.

no가 drain 되어야 하는 경우 node controller 또는 kubelet은 NoExecute effect를 추가한다. effect는 `node.kubernetes.io/not-ready`, `node.kubernetes.io/unreachable` taint에 추가된다. 장애 상태가 정상으로 돌아오면 kubelet 또는 no controller가 관련 taint를 제거한다.

no에 unreachable이면 kube-apiserver가 no의 kubelet과 통신에 실패할 수 있다. 그렇기 때문에 po에 대한 삭제를 kubelet에 통보할 수 없다. 그렇기 때문에 해당 no에 po는 계속해서 실행중일 수 있다.

> **Note**:  
> control plane은 no에 새 taint를 추가하는 비율을 제한한다. 이 비율-제한은 많은 no가 동시에 unreachable이 될 때(예를 들어, 네트워크 중단으로) 트리거될 eviction 개수를 관리한다.

po의 `.spec.tolerations[*].tolerationSeconds` 필드를 사용해 failing or unresponsive no에서 eviction 되기 전의 시간을 설정할 수 있다.

예를 들어, 로컬 상태가 많은 애플리케이션은 네트워크 분할의 장애에서 네트워크가 복구된 후에 po가 축출되는 것을 피하기 위해 오랫동안 no에 binding된 상태를 유지하려고 할 수 있다. 이 경우 po가 사용하는 toleration은 다음과 같다.
``` yaml
tolerations:
- key: "node.kubernetes.io/unreachable"
  operator: "Exists"
  effect: "NoExecute"
  tolerationSeconds: 6000
```

> **Note**: 
> k8s는 사용자가 명시적으로 설정하지 않았다면 자동으로 `node.kubernetes.io/not-ready` 와 `node.kubernetes.io/unreachable` 에 대해 tolerationSeconds=300 으로 toleration을 추가한다.
>
> 자동으로 추가된 이 toleration은 이러한 문제 중 하나가 감지된 후 5분 동안 po가 no에 binding된 상태를 유지함을 의미한다.

ds는 po 생성 시, 아래 taint에 대해 tolerationSeconds가 없는 NoExecute toleration을 추가한다.
- `node.kubernetes.io/unreachable`
- `node.kubernetes.io/not-ready`

이렇게 하면 이러한 문제로 인해 ds po가 eviction되지 않는다.

## Taint Nodes by Condition
control plane은 node controller를 이용해 [no condition](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/#node-conditions)에 대한 NoSchedule effect를 사용해 자동으로 taint를 추가한다.

scheduler는 스케줄링 결정을 내릴 때 no condition을 확인하는 것이 아니라 taint를 확인한다. 이렇게 하면 no condition이 스케줄링에 직접적인 영향을 주지 않는다. 예를 들어 DiskPressure no condition이 활성화된 경우 control plane은 `node.kubernetes.io/disk-pressure` taint를 추가하고 영향을 받는 no에 새 po를 할당하지 않는다. MemoryPressure no condition이 활성화되면 control plane이 `node.kubernetes.io/memory-pressure` taint를 추가한다.

새로 생성된 po에 po toleration을 추가해 no condition을 무시하도록 할 수 있다. 또한 control plane은 BestEffort 이외의 QoS class를 가지는 po에 `node.kubernetes.io/memory-pressure` toleration을 추가한다. 이는 k8s가 Guaranteed 또는 Burstable QoS class를 갖는 po(memory request가 설정되지 않은 po 포함)를 마치 그 po들이 memory pressure에 대처 가능한 것처럼 다루는 반면 새로운 BestEffort po는 영향을 받는 no에 할당하지 않기 때문이다.

ds controller는 다음의 NoSchedule toleration을 자동으로 추가하여, ds이 중단되는 것을 방지한다.
- `node.kubernetes.io/memory-pressure`
- `node.kubernetes.io/disk-pressure`
- `node.kubernetes.io/pid-pressure` (1.14 이상)
- `node.kubernetes.io/unschedulable` (1.10 이상)
- `node.kubernetes.io/network-unavailable` (host network만 해당)

이러한 toleration을 추가하면 이전 버전과의 호환성이 보장된다. ds에 임의의 toleration을 추가할 수도 있다.