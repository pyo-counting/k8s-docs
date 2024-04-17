po가 특정 no에서만 실행될 수 있도록 제약하거나 특정 no를 선호하도록 할 수 있다. 이를 위한 여러 가지 방법이 있으며 권장되는 접근 방식은 모두 label selector를 사용하는 것이다. 일반적으로 이러한 제약 조건을 설정할 필요가 없다. scheduler가 합리적인 배치를 자동으로 수행하기 때문이다(예: po를 no 간에 분산하여 no에 충분한 여유 resource가 없는 no에 po를 배치하지 않는다). 그러나 SSD가 연결된 no에 po가 배치되도록 하거나 많은 통신을 하는 두 가지 다른 서비스의 po를 동일한 availability zone에 배치하려는 경우와 같이 특정 no에 po를 배포하고자 하는 경우가 있을 수 있다.

k8s에서 특정 po를 배치할 위치(no)를 선택하는 데 다음과 같은 방법을 사용할 수 있다.
- po의 `.spec.nodeSelector` 필드
- po의 `.spec.affinity` 필드
- po의 `.spec.nodeName` 필드
- po의 `.spec.topologySpreadConstraints` 필드

## Node labels
다른 k8s object와 마찬가지로 no도 `.metadata.labels` 필드가 있다. k8s는 cluster의 모든 no에 [standard set of labels](https://kubernetes.io/docs/reference/node/node-labels/)을 할당한다(k8s 구성 요소 중 kubelet이 설정).

> **Note**:  
> label 값은 cloud provider마다 다를 수 있으며 보장되지 않는다ㅏ. 예를 들어, `kubernetes.io/hostname` label의 값은 일부 환경에서 no 이름과 동일할 수 있고 다른 환경에서는 다른 값을 가질 수도 있다.

### Node isolation/restriction
no에 label을 추가해 특정 no나 no 그룹에 po를 스케줄링할 수 있다. 이 기능을 사용해 특정 po가 특정 isolation, security, regulatory property을 가진 no에서만 실행되도록 할 수 있다.

no isolation를 위해 label을 사용하는 경우 kubelet이 수정할 수 없는 label key를 사용하는 것을 권장한다. 이를 통해 손상(compromised)된 no(kubelet)가 해당 label을 자체적으로 수정해 scheduler가 해당 no에 workload를 스케줄링하는 것읇 방지할 수 있다.

[`NodeRestriction` admission plugin](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#noderestriction)은 kubelet이 `node-restriction.kubernetes.io/` 접두사가 있는 label을 설정하거나 수정하는 것을 방지한다.

no 격리를 위해 해당 label 접두사를 사용하기 위해 아래 내용을 확인한다.
1.[Node authorizer](https://kubernetes.io/docs/reference/access-authn-authz/node/)을 사용하는지, `NodeRestriction` admission plugin이 활성화 됐는지
2. no에 `node-restriction.kubernetes.io/` 접두사가 있는 label을 추가하고 이러한 label을 label selector에서 사용한다. 예를 들어, `example.com.node-restriction.kubernetes.io/fips=true` 또는 `example.com.node-restriction.kubernetes.io/pci-dss=true`와 같이 사용한다.

## nodeSelector
no 제약 조건 중 가장 간단한 방법은 po의 `.spec.nodeSelector` 필드를 사용하는 것이다. k8s는 해당 필드에 명시된 label 조건을 만족하는 no에만 po를 스케줄링한다.

자세한 내용은 [Assign Pods to Nodes](https://kubernetes.io/docs/tasks/configure-pod-container/assign-pods-nodes)를 참고한다.

## Affinity and anti-affinity
nodeSelector는 po를 특정 label이 있는 no에 스케줄링 되도록 제한하는 가장 간단한 방법이다. affinity, anti-affinity은 제한에 대한 더 많은 기능을 제공한다. affinity, anti-affinity의 이점은 아래와 같다.
- affinity/anti-affinity가 더 표현적(expressive)이다. nodeSelector는 명시된 label이 있는 no만 선택할 수 있다. affinity/anti-affinity는 선택 뿐만 아니라 더 많은 제어를 제공한다.
- 규칙이 "soft", "preference" 임을 나타낼 수 있으며 scheduler는 매칭되는 no를 찾지 못할 경우에도 po를 스케줄링할 수 있다.
- no label 대신 no(또는 다른 topological domain)에서 실행 중인 다른 po의 label을 사용해 po를 제한할 수 있다. 이를 통해 po가 동일 no에 함께 배치되도록 규칙을 정의할 수 있다.

affinity 기능은 다음의 두 가지 종류로 구성된다:
- `node affinity`: nodeSelector 필드와 비슷하지만 더 표현적이고 소프트(soft) 규칙을 지정할 수 있다.
- `inter-pod affinity/anti-affinity`: 다른 po의 label을 이용하여 po를 제한할 수 있다.

### Node affinity
node affinity는 개념적으로 nodeSelector 와 비슷하며 no의 label을 기반으로 po가 스케줄링될 수 있는 no를 제한할 수 있다. node affinity에는 다음 두 종류가 있다.
- `requiredDuringSchedulingIgnoredDuringExecution`: 규칙이 만족되지 않으면 scheduler가 po를 스케줄링할 수 없다. 이 기능은 nodeSelector와 유사하지만, 좀 더 표현적인 문법을 제공한다.
- `preferredDuringSchedulingIgnoredDuringExecution`: scheduler는 조건을 만족하는 no를 찾으려고 노력한다. 매칭되는 no가 없더라도 scheduler는 여전히 po를 스케줄링한다.

> **Note**:  
> 두 유형에서, `IgnoredDuringExecution`는 k8s가 po를 스케줄링한 뒤에 no의 label이 변경되어도 po는 계속 해당 no에서 실행됨을 의미한다.

po의 `.spec.affinity.nodeAffinity` 필드에 node affinity를 설정할 수 있다.

아래는 예시다.
``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-node-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: topology.kubernetes.io/zone
            operator: In
            values:
            - antarctica-east1
            - antarctica-west1
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: another-node-label-key
            operator: In
            values:
            - another-node-label-value
  containers:
  - name: with-node-affinity
    image: registry.k8s.io/pause:2.0
```

위 예시는 아래 규칙이 적용된다.
- no는 반드시 `topology.kubernetes.io/zone` key의 값이 antarctica-east1 또는 antarctica-west1인 label이 존재해야 한다.
- no는 `another-node-label-key` key의 값이 another-node-label-value인 label이 있으면 선호된다.

operator 필드에는 In, NotIn, Exists, DoesNotExist, Gt, Lt를 사용할 수 있다.

NotIn, DoesNotExist는 node anti-affinity를 정의할 떄 허용된다. 이 대신 [node taints](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)를 사용할 수도 있다.

> **Note**:  
> - nodeSelector, nodeAffinity를 모두 사용한다면 스케줄링이 되기 위해 두 조건을 모두(AND) 만족해야 한다.
> - nodeAffinity에 대해 nodeSelectorTerms를 여러개 사용할 경우 명시된 nodeSelectorTerms 중 하나(OR)를 만족하는 no에도 po가 스케줄링 될 수 있다.
> - nodeSelectorTerms에 대해 matchExpressions를 여러개 사용하는 경우 모든(AND) matchExpressions를 만족하는 no에만 po가 스케줄링 될 수 있다.

#### Node affinity weight
preferredDuringSchedulingIgnoredDuringExecution affinity 타입의 경우 weight 필드를 1~100 사이의 값으로 설정할 수 있다. scheduler가 모든 po 스케줄링 required 규칙을 만족하는 no를 찾으면 scheduler는 추가적으로 no가 만족하는 모든 preferred 규칙의 weight 값을 더한다.

더해진 최종 값은 해당 no에 대한 우선 순위 함수 점수에 더해진다. scheduler가 po에 대한 스케줄링 판단을 할 때, 총 점수가 가장 높은 no가 우선 순위를 갖는다.

아래는 예시다.
``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-affinity-anti-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/os
            operator: In
            values:
            - linux
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: label-1
            operator: In
            values:
            - key-1
      - weight: 50
        preference:
          matchExpressions:
          - key: label-2
            operator: In
            values:
            - key-2
  containers:
  - name: with-node-affinity
    image: registry.k8s.io/pause:2.0
```

위 예시에 대해서 requiredDuringSchedulingIgnoredDuringExecution 규칙을 만족하는 no가 2개 있다고 가정한다. 1개의 no는 label-1:key-1 label이 있고 다른 개의 no에는 label-2:key-2가 있으면, scheduler는 각 no의 weight를 확인한 뒤 해당 no 각각 점수에 weight를 더하고, 최종 점수가 가자 높은 no에 po를 스케줄링한다.

> **Note**:  
> 위 예시에서 k8s가 정상적으로 po를 스케줄링하기 위해, no는 kubernetes.io/os=linux label이 반드시 있어야 한다.

### Node affinity per scheduling profile
여러 [scheduling profile](https://kubernetes.io/docs/reference/scheduling/config/#multiple-profiles)을 구성할 때 profile에 대해 node affinity를 설정할 수 있는데, 이는 profile이 특정 no 집합에만 적용되는 경우 유용하다. 이를 위해 [scheduler configuration](https://kubernetes.io/docs/reference/scheduling/config/) 내 [`NodeAffinity` plugin](https://kubernetes.io/docs/reference/scheduling/config/#scheduling-plugins)의 arg 필드에 addedAffinity을 추가한다.
``` yaml
apiVersion: kubescheduler.config.k8s.io/v1beta3
kind: KubeSchedulerConfiguration

profiles:
  - schedulerName: default-scheduler
  - schedulerName: foo-scheduler
    pluginConfig:
      - name: NodeAffinity
        args:
          addedAffinity:
            requiredDuringSchedulingIgnoredDuringExecution:
              nodeSelectorTerms:
              - matchExpressions:
                - key: scheduler-profile
                  operator: In
                  values:
                  - foo
```

addedAffinity는 .spec.schedulerName을 foo-scheduler로 설정하는 모든 po에 적용되며 PodSpec에 지정된 NodeAffinity도 적용된다. 즉, 이 po를 실행하기 위해 no는 addedAffinity, .spec.NodeAffinity를 모두 충족해야 한다.

addedAffinity는 엔드 유저에게 표시되지 않으므로, 예상치 못한 동작이 일어날 수 있다. scheduler profile 이름과 명확한 상관 관계가 있는 no label을 사용해야 한다.

> **Note**:  
> ds po를 생성하는 ds controller는 scheduling profile을 지원하지 않는다. ds controller가 po를 생성할 때, 기본 k8s scheduler는 해당 po를 배치하고 ds controller의 모든 nodeAffinity 규칙을 준수한다.



### Inter-pod affinity and anti-affinity
po 사이에 affinity와 anti-affinity를 사용해 no label 대신, 각 no에 이미 실행 중인 다른 po의 label을 기반으로 po가 스케줄링될 no를 제한할 수 있다.

po 사이에 affinity와 anti-affinity 규칙은 "X가 규칙 Y를 충족하는 po를 이미 실행중인 경우 이 po는 X에서 실행해야 한다(anti-affinity의 경우에는 "실행하면 안 된다")"의 형태이며, 여기서 X는 노드, rack, 클라우드 제공자 zone 또는 region 등이며 Y는 k8s가 충족할 규칙이다.

이러한 규칙(Y)은 label selector로 작성하며 연관된 ns 목록을 선택적으로 명시할 수도 있다. k8s에서 po는 ns에 속하는(namespaced) object이므로, po label도 암묵적으로 특정 ns에 속하게 된다. po label에 대한 모든 label selector는 k8s가 해당 label을 어떤 ns에서 탐색할지를 명시해야 한다.

topologyKey를 사용하여 토폴로지 도메인(X)를 나타낼 수 있으며, 이는 시스템이 도메인을 표시하기 위해 사용하는 노드 label의 키이다. 이에 대한 예시는 [Well-Known Labels, Annotations and Taints](https://kubernetes.io/docs/reference/labels-annotations-taints/)를 참고한다.

**Note**: po 사이에 affinity와 anti-affinity에는 상당한 양의 프로세싱이 필요하기에 대규모 클러스터에서는 스케줄링 속도가 크게 느려질 수 있다. 수백 개의 no를 넘어가는 클러스터에서 이를 사용하는 것은 추천하지 않는다.

**Note**: po anti-affinity에서는 no에 일관된 label을 지정해야 한다. 즉, 클러스터의 모든 no는 topologyKey 와 매칭되는 적절한 레이블을 가지고 있어야 한다. 일부 또는 모든 no에 topologyKey로 명시한 label이 없는 경우에는 의도하지 않은 동작이 발생할 수 있다.

#### Types of inter-pod affinity and anti-affinity
no affinity와 마찬가지로 po affinity, anti-affinity에는 다음의 2 종류가 있다:

- requiredDuringSchedulingIgnoredDuringExecution
- preferredDuringSchedulingIgnoredDuringExecution

예를 들어, requiredDuringSchedulingIgnoredDuringExecution affinity를 사용하여 서로 통신을 많이 하는 두 파드를 동일 클라우드 제공자 zone에 배치하도록 scheduler에게 지시할 수 있다. 비슷하게, preferredDuringSchedulingIgnoredDuringExecution anti-affinity를 사용해 po를 여러 클라우드 제공자 zone에 퍼뜨릴 수 있다.

po사이의 affinity를 사용하려면, po 스펙에 affinity.podAffinity 필드를 사용한다. po간 anti-affinity를 사용하려면, po 스펙에 affinity.podAntiAffinity 필드를 사용한다.

#### Pod affinity example
``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-pod-affinity
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: security
            operator: In
            values:
            - S1
        topologyKey: topology.kubernetes.io/zone
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: security
              operator: In
              values:
              - S2
          topologyKey: topology.kubernetes.io/zone
  containers:
  - name: with-pod-affinity
    image: registry.k8s.io/pause:2.0
```

이 예시는 하나의 po affinity 규칙과 하나의 po anti-affinity 규칙을 정의한다. po affinity 규칙은 "하드" requiredDuringSchedulingIgnoredDuringExecution을, anti-affinity 규칙은 "소프트" preferredDuringSchedulingIgnoredDuringExecution을 사용한다.

affinity 규칙은 security=S1 label이 있는 하나 이상의 기존 po의 zone와 동일한 zone에 있는 no에만 po를 스케줄링하도록 scheduler에 지시한다. 더 정확히 말하면, 만약 security=S1 po label이 있는 po를 실행하고 있는 no가 zone=V에 하나 이상 존재한다면, scheduler는 po를 topology.kubernetes.io/zone=V label이 있는 no에 배치해야 한다.

anti-affinity 규칙은 security=S2 label이 있는 하나 이상의 기존 po의 zone와 동일한 zone에 있는 no에는 가급적 파드를 스케줄링하지 않도록 scheduler에 지시한다. 더 정확히 말하면, 만약 security=S2 po label이 있는 po가 실행되고 있는 zone=R에 다른 no도 존재한다면, scheduler는 security=S2 label이 있는 no에는 가급적 해당 po를 스케줄링하지 않야아 한다.

po affinity, anti-affinity에 대한 자세한 내용은 [design proposal](https://github.com/kubernetes/design-proposals-archive/blob/main/scheduling/podaffinity.md)을 참조한다. 

po affinity, anti-affinity의 operator 필드에 In, NotIn, Exists 및 DoesNotExist 값을 사용할 수 있다.

원칙적으로, topologyKey에는 성능과 보안상의 이유로 다음의 예외를 제외하면 어느 label 키도 사용할 수 있다.

- po affinity, anti-affinity에 대해, 빈 topologyKey 필드는 requiredDuringSchedulingIgnoredDuringExecution, preferredDuringSchedulingIgnoredDuringExecution 내에 허용되지 않는다.
- requiredDuringSchedulingIgnoredDuringExecution po anti-affinity 규칙에 대해, LimitPodHardAntiAffinityTopology admission controller는 topologyKey를 kubernetes.io/hostname으로 제한한다. 커스텀 토폴로지를 허용하고 싶다면 admission controller를 수정하거나 비활성화할 수 있다.

labelSelector와 topologyKey에 더하여 선택적으로, labelSelector가 비교해야 하는 ns의 목록을 labelSelector 및 topologyKey 필드와 동일한 계층 namespaces 필드에 명시할 수 있다. 생략하거나 비워 두면, 해당 affinity, anti-affinity 정의가 있는 po의 ns를 기본값으로 사용한다.

#### Namespace selector
ns 집합에 대한 label 쿼리인 namespaceSelector 를 사용해 일치하는 ns를 선택할 수도 있다. affinity는 namespaceSelector와 namespaces에 의해 선택된 모든 ns에 적용된다. 빈 namespaceSelector ({})는 모든 ns와 일치하는 반면, null 또는 빈 namespaces 목록과 null namespaceSelector 는 규칙이 적용된 po의 ns에 매치된다.

#### More practical use-cases
po affinity, anti-affinity는 rs, sts, deploy등과 함께 사용할 때 더욱 유용할 수 있다. 이러한 규칙을 사용하여, workload 집합이 예를 들면 '동일한 node'와 같이 동일하게 정의된 토폴로지와 같은 위치에 배치되도록 쉽게 구성할 수 있다.

redis와 같은 in-memory 캐시를 사용하는 웹 애플리케이션을 실행하는 세 개의 노드로 구성된 클러스터를 가정한다. 이 때 웹 서버를 가능한 한 캐시와 같은 위치에서 실행되도록 하기 위해 po affinity, anti-affinity를 사용할 수 있다.

다음의 redis 캐시 deploy 예시에서, replica는 app=store label을 갖는다. podAntiAffinity 규칙은 scheduler로 하여금 app=store label이 있는 replica를 한 no에 여러 개 배치하지 못하도록 한다. 이렇게 하여 캐시 po를 각 노드에 분산하여 생성한다.

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-cache
spec:
  selector:
    matchLabels:
      app: store
  replicas: 3
  template:
    metadata:
      labels:
        app: store
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - store
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: redis-server
        image: redis:3.2-alpine
```

아래 deploy는 app=web-store label을 갖는 replica를 생성한다. po affinity 규칙은 scheduler로 하여금 app=store label이 있는 po를 실행 중인 no에 각 replica를 배치하도록 한다. po anti-affinity 규칙은 scheduler로 하여금 app=web-store label이 있는 서버 po를 한 no에 여러 개 배치하지 못하도록 한다.

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-server
spec:
  selector:
    matchLabels:
      app: web-store
  replicas: 3
  template:
    metadata:
      labels:
        app: web-store
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - web-store
            topologyKey: "kubernetes.io/hostname"
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - store
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: web-app
        image: nginx:1.16-alpine
```

위의 두 deploy를 생성하면 다음과 같은 클러스터 형상이 나타나는데, 세 no에 각 웹 서버가 캐시와 함께 있도록 배치된다.

|node-1|node-2|node-3|
|------|------|------|
|webserver-1|webserver-2|webserver-3|
|cache-1|cache-2|cache-3|

전반적인 효과는 동일한 no에서 실행 중인 단일 클라이언트가 각 캐시 인스턴스에 접근할 가능성이 있다는 것이다. 이 접근 방식은 skew(불균형 로드)와 대기 시간을 모두 최소화하는 것을 목표로 한다.

[ZooKeeper tutorial](https://kubernetes.io/docs/tutorials/stateful-application/zookeeper/#tolerating-node-failure)에서 위 예시와 동일한 기술을 사용해 고 가용성을 위한 anti-affinity로 구성된 sts의 예시를 확인할 수 있다.

## nodeName
spec.nodeName은 affinity, spec.nodeSelector보다 더 직접적인 no 선택 방법이다. spec.nodeName 필드가 명시되면 scheduler는 po를 무시하고 명시된 no의 kubelet이 해당 po를 자기 no에 배치하려고 시도한다. 이는 다른 규칙보다 우선 적용된다.

몇 가지 제한 사항이 있다:

- 명시한 no가 존재하지 않을 경우, po는 실행되지 않고 삭제될 수도 있다.
- 명시된 no에 po를 수용할 수 있는 리소스가 없는 경우 po는 실패하고 이에 대한 이유는 다음과 같이 표기된다: OutOfmemory 또는 OutOfcpu
- 클라우드 환경에서 no의 이름은 항상 예측할 수 없다.

다음은 예시다:

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
  nodeName: kube-01
```

## Pod topology spread constraints
topology spread constraint를 사용해 regions, zones, nodes, 사용자 정의 topology domain 간에 po가 클러스터 전체에 분산되는 방식을 제어할 수 있다. 이를 통해 성능, 예상 가용성, 전체 활용도를 향상시킬 수 있다.