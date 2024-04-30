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
- `maxSkew`: po가 고르지 않게 분포될 수 있는 정도를 나타낸다. 이 필드는 필수이며 0보다 큰 숫자를 사용해야 한다. 이 필드의 의미는 `whenUnsatisfiable` 필드 값에 따라 바뀐다.
    - `whenUnsatisfiable: DoNotSchedule`: 
    - `whenUnsatisfiable: ScheduleAnyway`:
- `minDomains`: