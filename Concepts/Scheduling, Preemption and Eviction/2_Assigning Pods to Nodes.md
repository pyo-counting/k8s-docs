특정 no에 po를 실행할 수 있도록 제한할 수 있다. 이를 수행하는 방법에는 여러 가지가 있으며 권장되는 접근 방식은 label selector를 사용해 선택을 용이하게 하는 것이다. 보통은 scheduler가 자동으로 합리적인 배치(예: 리소스가 부족한 no에 po를 배치하지 않도록 no 간에 po를 분배)를 수행하기 때문에 이러한 제약 조건은 필요하지 않다. 그러나, 예를 들어 SSD가 장착된 머신에 po가 배포되도록 하거나 또는 많은 통신을 하는 두 개의 서로 다른 서비스의 po를 동일한 가용성 영역(availability zone)에 배치하는 경우와 같이, po가 어느 no에 배포될지를 제어해야 하는 경우도 있다.

k8s가 특정 po를 어느 no에 스케줄링할지 아래 설정을 통해 결정할 수 있다:

- node label에 매칭되는 nodeSelector
- affinity, anti-affinity
- nodeName 필드
- pod topology spread constraints

## Node labels
다른 k8s object와 마찬가지로 no는 label을 갖는다. 직접 label을 추가/삭제/수정 할 수 있다. k8s는 클러스터 내 모든 no에 기본적으로 필요한 label을 할당한다. 공통적인 no label은 [Well-Known Labels, Annotations and Taints](https://kubernetes.io/docs/reference/labels-annotations-taints/) 페이지를 참고한다.

**Note**: 이러한 label의 값은 클라우드 제공자에 따라 다르며 신뢰성이 보장되지 않는다. 예를 들어 kubernetes.io/hostname의 값은 일부 환경에서는 no 이름과 동일하고 다른 환경에서는 다른 값을 사용할 수도 있다.

### Node isolation/restriction
no에 label을 추가해 po를 특정 no 또는 no 그룹에 스케줄링되도록 할 수 있다. 이 기능을 사용해 특정 po가 특정 격리/보안/규제 속성을 만족하는 no에서만 실행되도록 할 수 있다.

no 격리를 위해 label을 사용할 때 kubelet이 변경할 수 없는 label 키를 설정해야 한다. 이를 통해 손상된 no가 해당 label을 자체적으로 수정해 scheduler가 해당 no에 workload를 스케줄링하는 것읇 방지할 수 있다.

NodeRestriction admission plugin은 kubelet이 node-restriction.kubernetes.io/ 접두사를 갖는 label을 설정하거나 변경하지 못하도록 한다.

no 격리를 위해 label 접두사를 사용하려면:

1. node authorizer를 사용하고 있는지, 그리고 NodeRestriction admission plugin을 활성화 했는지 확인한다.
2. no에 node-restriction.kubernetes.io/ 접두사를 갖는 label을 추가하고, node selector에서 해당 abel을 사용한다. 예: example.com.node-restriction.kubernetes.io/fips=true 또는 example.com.node-restriction.kubernetes.io/pci-dss=true

## nodeSelector
spec.nodeSelector 필드는 no 선택 제한의 가장 간단하면서도 추천하는 방법이다. k8s는 사용자가 해당 필드에 명시한 label을 갖는 no에만 po를 스케쥴링한다.

## Affinity and anti-affinity

### Node affinity

### Node affinity weight

### Node affinity per scheduling profile
nodeSelector는 po를 특정 label이 있는 no에 스케줄링 되도록 제한하는 가장 간단한 방법이다. affinity, anti-affinity은 제한에 대한 더 많은 기능을 제공한다. affinity, anti-affinity의 이점은 아래와 같다:

- affinity/anti-affinity가 더 표현적(expressive)이다. nodeSelector는 명시된 label이 있는 no만 선택할 수 있다. affinity/anti-affinity는 선택 뿐만 아니라 더 많은 제어를 제공한다.
- 