topology spread constraint을 사용해 failure-domain(region, zone, no, 사용자 정의 topology domain) 사이에 po가 분포되는 방법을 제어할 수 있다. 이를 통해 고가용성, 효율적인 리소스 사용을 달성할 수 있다.

kube-scheduler의 PodTopologySpread plugin을 이용해 cluster-level의 기본 constraints를 설정할 수도 있다.

## Motivation
20개의 no가 있는 cluster에서 wokrload의 replica가 자동 scaling 되도록 하고싶을 수 있다. workload에 대한 2개 po replica가 단일 no에 실행되는 것을 원하지 않을 수 있다.

위와 같은 기본적인 사용 말고도 고가용성, cluster 사용성에 대한 이점을 얻기 위해 사용할 수 있다.

po를 많이 실행할수록 신경써야할 부분이 많다. 3개의 no에 각각 5개의 po를 실행한다고 가정하자. no는 po를 실행할 충분한 여유가 있지만 이 workload와 상호작용하는 client는 3개의 서로 다른 datacenter(또는 infrastructure zone)에 분산되어 있다. 단일 no 장애에 대한 걱정은 없지만 latency가 원하는 것보다 더 높을 수 있으며 서로 다른 zone 트래픽 전송에 따른 네트워크 비용을 지불하게 된다.

정상적인 운영을 위해 각 infrastructure zone에 비슷한 수의 replica를 스케줄링하고 문제가 발생한 경우 cluster를 자가 복구하도록 하는 것이 좋다.

po의 topology spread constraint은 이러한 설정을 위한 declarative 방법을 제공한다.

## `topologySpreadConstraints` field
po는 `.spec.topologySpreadConstraints`을 지원한다.
``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: example-pod
spec:
  # Configure a topology spread constraint
  topologySpreadConstraints:
    - maxSkew: <integer>
      minDomains: <integer> # optional
      topologyKey: <string>
      whenUnsatisfiable: <string>
      labelSelector: <object>
      matchLabelKeys: <list> # optional; beta since v1.27
      nodeAffinityPolicy: [Honor|Ignore] # optional; beta since v1.26
      nodeTaintsPolicy: [Honor|Ignore] # optional; beta since v1.26
  ### other Pod fields go here
```

### Spread constraint definition
`.spec.topologySpreadConstraints`에 1개 이상의 목록을 정의해 kube-scheduler가 cluster에서 어떻게 po를 분배할지에 대해 정의한다. 필드는 다음과 같다.
- `topologyKey`: (required) no label key. 해당 label key를 갖고 동일한 값을 갖는 no는 동일 topology로 간주된다. 각 topology instance(no의 label key&value가 같은 집합)를 domain이라고 부른다. 그리고 domain을 구성하는 no가 `nodeAffinityPolicy`, `nodeTaintsPolicy` 요구 사항을 충족(요구 사항이 없는 경우에도)하는 경우 eligible domain이라고 한다. kube-scheduler는 각 domain에 po를 균등하게 배포하려고 한다. 
- `minDomains`: (optional) eligible domain의 최소 개수. domain의 개수가 `minDomains`보다 작으면 global minimum이 0이 되기 때문에 각 domain에는 최대 `maxSkew` 개수의 po만 스케줄링될 수 있다.
  > **Note**:  
  > k8s v1.30 이전에서 `minDomains` 필드는 MinDomainsInPodTopologySpread feature gate(v1.28부터 기본 활성화)가 활성화된 경우에만 사용할 수 있다. 이 전 버전에서는 비활성화 됐기 때문에 기본적으로 사용이 불가하다.
  - 값은 0보다 커야하며 `whenUnsatisfiable: DoNotSchedule`일 경우에만 사용할 수 있다.
  - topology key와 매칭되는 eligible domain 개수가 `minDomains`보다 작으면 global minimum을 0으로 간주해 skew를 계산한다(global minimum은 eligible domain에서 매칭되는 po의 최소 개수).
  - topology key와 매칭되는 eligible domain 개수가 `minDomains`와 같거나 더 크면 scheduling에 영향을 주지 않는다.
  - `minDomains`을 명시하지 않으면 값이 1인 것으로 간주된다.
- `maxSkew`: (required) po가 고르지 않게 분포될 수 있는 정도를 나타낸다. 이 필드는 필수이며 0 보다 큰 숫자를 사용해야 한다. 이 필드의 의미는 `whenUnsatisfiable` 필드 값에 따라 바뀐다.
    - `whenUnsatisfiable: DoNotSchedule`: `maxSkew`는 대상 topology에 있는 매칭 po의 갯수와 global minimum(eligible domain 중 매칭되는 po 개수 값 중 가장 작은 수, 만약 eligible domain이 `minDomains`보다 작으면 0으로 간주) 사이의 최대 허용 차이를 정의한다. 예를 들어 2, 2, 1개의 매칭 po를 갖는 3개 zone이 있는 경우 `maxSkew`가 1이라면 global minimum은 1이다.
    - `whenUnsatisfiable: ScheduleAnyway`: kube-scheduler는 skew를 줄이기 위해 도움이되는 topology에 더 높은 우선 순위를 부여한다.
- `whenUnsatisfiable`: (required) spread constraint를 만족하지 않는 po를 처리할 방법을 설정한다.
  - `DoNotSchedule`: (default) 스케줄링을 수행하지 않는다.
  - `ScheduleAnyway`: skew를 최소화하는 no의 우선순위를 지정해 스케줄링을 계속 수행하도록 한다.
- `labelSelector`: 매칭 po를 찾는데 사용된다. label selector에 매칭되는 po는 topology domain에 존재하는 po 개수를 계산하는데 사용된다.
- `matchLabelKeys`: `labelSelector`와 마찬가지로 매칭 po를 찾는데 사용된다. 다만 label key의 유무만 검사한다. `labelSelector` 필드를 사용하는 경우에만 사용할 수 있으며 `labelSelector`와 중복되는 key를 사용할 수 없다. 그리고 `labelSelector`의 label 목록과 AND 연산 결과 label에 매칭되는 po를 찾는다. 빈 값은 모든 po를 의미하며 `labelSelector`만 고려한다.   
`matchLabelkey`를 이용하면 서로 다른 revision에 따라 `pod.spec`을 변경할 필요 없다. 예를 들어 deploy를 사용하는 경우 deploy가 추가하는 기본 [pod-template-hash](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#pod-template-hash-label) label을 사용해 서로 다른 revision을 구분할 수 있다.
  ``` yaml
      topologySpreadConstraints:
          - maxSkew: 1
            topologyKey: kubernetes.io/hostname
            whenUnsatisfiable: DoNotSchedule
            labelSelector:
              matchLabels:
                app: foo
            matchLabelKeys:
              - pod-template-hash
  ```
  > **Note**:  
  > matchLabelKeys는 1.27 버전부터 기본 활성화되며 beta-level 필드이다. MatchLabelKeysInPodTopologySpread feature gate를 비활성화해 해당 필드를 비활성화할 수 있다.
- `nodeAffinityPolicy`: topology spread skew를 계산할 때 po의 node affinity, node selector를 어떻게 처리할지 설정한다. 기본 값은 `Honor`다.
  - `Honor`: node affinity, node selector 결과 no에 대한 skew를 계산한다.
  - `Ignore`: 모든 no에 대한 skew를 계산한다.
  > **Note**:  
  > nodeAffinityPolicy는 1.26 버전부터 기본 활성화되며 beta-level 필드이다. NodeInclusionPolicyInPodTopologySpread feature gate를 비활성화해 해당 필드를 비활성화할 수 있다.
- `nodeTaintsPolicy`: topology spread skew를 계산할 때 no의 taints를 어떻게 처리할지 설정한다. 기본 값은 `Ignore`다.
  - `Honor`: taints가 없는 no와 tainted no지만 po의 toleration이 있는 no에 대해 skew를 계산한다.
  - `Ignore`: 모든 no에 대한 skew를 계산한다.
    > **Note**:  
  > nodeTaintsPolicy는 1.26 버전부터 기본 활성화되며 beta-level 필드이다. NodeInclusionPolicyInPodTopologySpread feature gate를 비활성화해 해당 필드를 비활성화할 수 있다.

po가 여러 `topologySpreadConstraint`를 사용하면 constraint는 논리 AND 연산자를 수행한다. kube-scheduler는 모든 constraint를 만족하는 no를 찾기위해 노력한다.

### Node labels
topology spread constraints는 topology domain을 식별하기 위해 각 no의 label에 의존한다.
``` yaml
  region: us-east-1
  zone: us-east-1a
```

> **Note**:  
> 위 예시에서는 private label을 사용했지만 일반적으로 `topology.kubernetes.io/zone`, `topology.kubernetes.io/region`와 같은 [well-known](https://kubernetes.io/docs/reference/labels-annotations-taints/) label key를 사용하는 것을 권장한다.

아래는 위 label을 갖는 4개 no에 대한 kubectl -L node,zone 명령어 예시다.
``` sh
NAME    STATUS   ROLES    AGE     VERSION   LABELS
node1   Ready    <none>   4m26s   v1.16.0   node=node1,zone=zoneA
node2   Ready    <none>   3m58s   v1.16.0   node=node2,zone=zoneA
node3   Ready    <none>   3m17s   v1.16.0   node=node3,zone=zoneB
node4   Ready    <none>   2m43s   v1.16.0   node=node4,zone=zoneB
```

## Consistency
You should set the same Pod topology spread constraints on all pods in a group.

Usually, if you are using a workload controller such as a Deployment, the pod template takes care of this for you. If you mix different spread constraints then Kubernetes follows the API definition of the field; however, the behavior is more likely to become confusing and troubleshooting is less straightforward.

You need a mechanism to ensure that all the nodes in a topology domain (such as a cloud provider region) are labeled consistently. To avoid you needing to manually label nodes, most clusters automatically populate well-known labels such as kubernetes.io/hostname. Check whether your cluster supports this.

## Topology spread constraint examples
### Example: one topology spread constraint

### Example: multiple topology spread constraints

### Example: conflicting topology spread constraints

#### Interaction with node affinity and node selectors

### Example: topology spread constraints with node affinity

## Implicit conventions
몇 가지 내장된 규칙이 있다.
- 동일 ns의 po만 고려한다.
- scheduler는 `topologySpreadConstraints[*].topologyKey`에 명시된 모든 label을 갖는 no만 고려한다. 이는 다음을 의미한다.
  1. 제외된 no에 위치한 po는 maxskew 계산에 계산하지 않는다.
  2. 새롭게 생성될 po는 이미 제외된 no에 스케쥴링 될 수 없다.
- 새롭게 생성된 po의 `topologySpreadConstraints[*].labelSelector`가 해당 po의 label과 일치하지 않을 경우, 발생할 수 있는 상황에 유의해야 한다. 위 예시에서 새롭게 생성된 po에서 label을 제거하더라도 maxskew를 만족하기 때문에 zone b에 있는 no에 배치할 수 있다. However, after that placement, the degree of imbalance of the cluster remains unchanged - it's still zone A having 2 Pods labeled as foo: bar, and zone B having 1 Pod labeled as foo: bar. If this is not what you expect, update the workload's topologySpreadConstraints[*].labelSelector to match the labels in the pod template

## Cluster-level default constraints
cluster에 기본 topology spread constraints를 설정하는 것도 가능하다. 기본 topology spread constrains는 아래의 경우에만 적용된다.
- `.spec.topologySpreadConstraints` 필드를 사용하지 않은 po
- svc, rs, sts에 속한 po

기본 constraints는 scheduler의 PodTopologySpread plugin을 통해 설정 가능하다. constraints는 `labelSelector` 필드가 빈 값이어야 한다는 점을 제외하고 po의 `.spec.topologySpreadConstraints`와 동일하다. selector는 po가 속한 svc, rs, sts를 통해 계산된다.

아래는 설정 예시다.
``` yaml
apiVersion: kubescheduler.config.k8s.io/v1beta3
kind: KubeSchedulerConfiguration

profiles:
  - schedulerName: default-scheduler
    pluginConfig:
      - name: PodTopologySpread
        args:
          defaultConstraints:
            - maxSkew: 1
              topologyKey: topology.kubernetes.io/zone
              whenUnsatisfiable: ScheduleAnyway
          defaultingType: List
```

### Built-in default constraints
사용자가 cluster-level 기본 constraints를 설정하지 않는 경우 kube-scheduler는 아래 기본 topology constraints를 사용한다.
``` yaml
defaultConstraints:
  - maxSkew: 3
    topologyKey: "kubernetes.io/hostname"
    whenUnsatisfiable: ScheduleAnyway
  - maxSkew: 5
    topologyKey: "topology.kubernetes.io/zone"
    whenUnsatisfiable: ScheduleAnyway
```

Also, the legacy SelectorSpread plugin, which provides an equivalent behavior, is disabled by default.

> **Note**:  
> The PodTopologySpread plugin does not score the nodes that don't have the topology keys specified in the spreading constraints. This might result in a different default behavior compared to the legacy SelectorSpread plugin when using the default topology constraints.
>
> If your nodes are not expected to have both kubernetes.io/hostname and topology.kubernetes.io/zone labels set, define your own constraints instead of using the Kubernetes defaults.

기본 topology constraints를 사용하길 원하지 않을 경우 PodTopologySpread plugin 설정에 `defaultingType` 필드를 List, `defaultConstraints` 필드를 빈 값으로 설정한다.
``` yaml
apiVersion: kubescheduler.config.k8s.io/v1beta3
kind: KubeSchedulerConfiguration

profiles:
  - schedulerName: default-scheduler
    pluginConfig:
      - name: PodTopologySpread
        args:
          defaultConstraints: []
          defaultingType: List
```

## Comparison with podAffinity and podAntiAffinity

## Known limitations
- po가 no에서 삭제되는 상황에도 constraints가 만족되는 것을 보장하지 않는다. 예를 들어 deploy의 scale down으로 po의 분포가 불균등할 수 있다. 이 경우를 대비해 [Descedhuler](https://github.com/kubernetes-sigs/descheduler)를 사용할 수 있다.
- tainted no에 존재하는 po도 계산된다.
- scheduler는 해당 시점에 cluster에 존재하는 no만 고려한다.