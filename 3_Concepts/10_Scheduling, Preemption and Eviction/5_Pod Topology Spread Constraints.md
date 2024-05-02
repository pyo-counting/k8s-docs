topology spread constraint을 사용해 failure-domain(region, zone, no, 사용자 정의 topology domain) 사이에 po가 분포되는 방법을 제어할 수 있다. 이를 통해 고가용성, 효율적인 리소스 사용을 달성할 수 있다.

## Motivation
20개의 no가 있는 cluster에서 wokrload의 replica가 자동 scaling 되도록 하고싶을 수 있다. workload에 대한 2개 po replica가 단일 no에 실행되는 것을 원하지 않을 수 있다.

위와 같은 기본적인 사용 말고도 고가용성, cluster 사용성에 대한 이점을 얻기 위해 사용할 수 있다.

po를 많이 실행할수록 신경써야할 부분이 많다. 3개의 no에 각각 5개의 po를 실행한다고 가정하자. no는 po를 실행할 충분한 여유가 있지만 이 workload와 상호작용하는 client는 3개의 서로 다른 datacenter(또는 infrastructure zone)에 분산되어 있다. 단일 no 장애에 대한 걱정은 없지만 latency가 원하는 것보다 더 높을 수 있으며 서로 다른 zone 트래픽 전송에 따른 네트워크 비용을 지불하게 된다.

정상적인 운영을 위해 각 infrastructure zone에 비슷한 수의 replica를 스케줄링하고 문제가 발생한 경우 cluster를 자가 복구하도록 하는 것이 좋다.

po의 topology spread constraint은 이러한 설정을 위한 declarative 방법을 제공한다.

## `topologySpreadConstraints` field
po는 `.spec.topologySpreadConstraints`을 지원한다.
``` yaml
---
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
- `maxSkew`: po가 고르지 않게 분포될 수 있는 정도를 나타낸다. 이 필드는 필수이며 0 보다 큰 숫자를 사용해야 한다. 이 필드의 의미는 `whenUnsatisfiable` 필드 값에 따라 바뀐다.
    - `whenUnsatisfiable: DoNotSchedule`: `maxSkew`는 대상 topology에 있는 매칭 po의 갯수와 global minimum(eligible domain에 매칭되는 po 수 또는 eligible domain가 `minDomains`보다 작으면 0) 사이의 최대 허용 차이를 정의한다. 예를 들어 2, 2, 1개의 매칭 po를 갖는 3개 zone이 있는 경우 `maxSkew`가 1이라면 global minimum은 1이다.
    - `whenUnsatisfiable: ScheduleAnyway`: kube-scheduler는 skew를 줄이기 위해 도움이되는 topology에 더 높은 우선 순위를 부여한다.
- `minDomains`: (optional) eligible domain의 최소 개수. topology의 각 instance를 domain이라고 부른다. no selector에 매칭되는 no를 eligible domain은 이라고 한다.
  - 값은 0보다 커야하며 `whenUnsatisfiable: DoNotSchedule`일 경우에만 사용할 수 있다.
  - topology key와 매칭되는 eligible domain 개수가 `minDomains`보다 작으면 global minimum을 9으로 간주해 skew를 계산한다. global minimum은 eligible domain에서 매칭되는 po의 최소 개수 또는 eligible domain가 `minDomains`보다 작으면 0으로 간주된다.
  - topology key와 매칭되는 eligible domain 개수가 `minDomains`와 갖거다 더 크면 scheduling에 영향을 주지 않는다.
  - `minDomains`을 명시하지 않으면 값이 1인 것으로 간주된다.
- `topologyKey`: no label key. 해당 label key를 갖고 동일한 값을 가지면 동일 topology로 간주된다. topology의 각 instance를 domain이라고 부른다. kube-scheduler는 각 domain에 po를 균등하게 배포하려고 한다. 그리고 no가 `nodeAffinityPolicy`, `nodeTaintsPolicy` 요구사항을 충족하는 domain을 eligible domain이라고 한다.
- `whenUnsatisfiable`: spread constraint를 만족하지 않는 po를 처리할 방법을 설정한다.
  - `DoNotSchedule`: (default) 스케줄링을 수행하지 않는다.
  - `ScheduleAnyway`: skew를 최소화하는 no의 우선순위를 지정해 스케줄링을 계속 수행하도록 한다.
- `labelSelector`: 매칭 po를 찾는데 사용된다. label selector에 매칭되는 po는 topology domain의 po 개수를 계산하는데 사용된다.
- `matchLabelKeys`: 
matchLabelKeys는 스프레드를 계산할 포드를 선택하기 위한 포드 레이블 키 목록입니다. 이 키는 포드 레이블에서 값을 조회하는 데 사용되며, 해당 키 값 레이블은 labelSelector로 AND 처리되어 들어오는 포드에 대해 스프레드를 계산할 기존 포드 그룹을 선택합니다. 동일한 키가 matchLabelKeys와 labelSelector 모두에 존재하지 않도록 금지됩니다. labelSelector가 설정되지 않은 경우 matchLabelKeys를 설정할 수 없습니다. 포드 레이블에 존재하지 않는 키는 무시됩니다. null 또는 빈 목록은 레이블Selector와만 일치하는 것을 의미합니다.
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

- `nodeAffinityPolicy`:
- `nodeTaintsPolicy`:

po가 여러 `topologySpreadConstraint`를 사용하면 constraint는 논리 AND 연산자를 수행한다. kube-scheduler는 모든 constraint를 만족하는 no를 찾기위해 노력한다.

### Node labels

## Consistency