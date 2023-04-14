k8s에서 hpa는 workload resource(deploy, sts, rs, rc)를 자동으로 업데이트하며, workload의 replica를 수요에 맞게 자동으로 scaling하는 것을 목표로 한다.

horizontal scaling은 부하 증가에 맞춰 po를 더 배포하는 것을 뜻한다. 이는 vertical scaling(k8s에서는 실행 중인 po에 더 많은 자원(메모리 또는 CPU 등)을 할당하는 것)과는 다르다.

부하가 줄어들고 po의 수가 설정한 최소 값보다 많을 경우 hpa는 workload resourc에게 scale down을 지시한다.

hpa는 scale 조절이 불가능한 object(예를 들어 ds)에는 적용할 수 없다.

hpa는 k8s API resource와 controller로 구현됐다. hpa API resource는 controller의 동작을 정의한다. k8s control plane 내에서 실행되는 hpa controller는 평균 CPU 사용률, 평균 메모리 사용률, 또는 다른 custom metric 등의 관측된 metric을 목표에 맞추기 위해 workdload의 replica 크기를 주기적으로 조정한다.

## How does a HorizontalPodAutoscaler work?
![How does a HorizontalPodAutoscaler work?](https://d33wubrfki0l68.cloudfront.net/4fe1ef7265a93f5f564bd3fbb0269ebd10b73b4e/1775d/images/docs/horizontal-pod-autoscaler.svg)

k8s는 hpa를 간헐적으로(intermittently) 실행되는 컨트롤 루프 형태로 구현했다(지속적인 프로세스가 아니다). 실행 주기는 kube-controller-manager의 --horizontal-pod-autoscaler-sync-period 파라미터에 의해 설정된다(기본 주기는 15초).

`.spec.metrics[*].type`은 metric 소스의 타입을 나타낸다. 사용 가능한 값은 ContainerResource, External, Object, Pods, Resource가 있다. ContainerResource은 HPAContainerMetrics feature-gate가 활성화 됐을 때 사용 가능하다.

- ContainerResource: scale 대상 po 내 container의 리소스(resource request, limit) metric.
- External: k8s object와 상관 없는 global metric. 클러스터 밖에서 실행되는 구성 요소에 대한 metric을 기반으로 autoscaling을 가능하게 한다. 예를 들어 클러스터 밖에 있는 로드밸런서의 QPS
- Object: 단일 k8s object을 나타내는 metric. 예를 들어 ingress에 대한 hits-per-second
- Pods: target po를 나타내는 metric. 예를 들어 transactions-processed-per-second
- Resource: scale 대상 po의 resource metric

각 주기마다, controller manager는 각 hpa 정의에 지정된 metric에 대한 리소스 사용률을 질의한다. controller manager는 `.spec.scaleTargetRef`에 정의된 타겟 resource를 찾고, 타겟 resource의 `.spec.selector` label을 보고 po를 선택하며, 리소스 metric API(po 단위 리소스 metric) 또는 custom metric API(그 외 모든 meric 용)로부터 metric을 수집한다.

- po 단위 리소스 metric(예를 들어 CPU)의 경우 controller는 hpa가 대상으로하는 각 po에 대한 metric을 리소스 metric API에서 가져온다. 목표 사용률 값이 설정된 경우 controller는 각 po의 container에 대한 해당 resource request의 백분율로 사용률 값을 계산한다. 목표 정수값이 설정된 경우 metric 값을 계산없이 직접 사용한다. 그리고나서 controller는 모든 대상 po에서 사용된 사용률의 평균 또는 정수 값(지정된 대상 유형에 따라 다름)을 가져와서 desired replica 값을 scale하는데 사용되는 비율을 계산한다.

  po의 일부 container에 resource request가 설정되지 않은 경우 po의 CPU 사용률은 정의되지 않으며 hpa는 해당 metric에 대해 아무런 조치도 취하지 않는다. autoscaling 알고리즘의 작동 방식에 대한 자세한 내용은 아래에서 살펴본다.

- po 단위 사용자 custom metric의 경우, controller는 사용률 값이 아닌 정수 값을 사용한다는 점을 제외하고는 po 단위 리소스 metric과 유사하게 동작한다.

- object metric, external metric의 경우, object를 표현하는 단일 metric을 가져온다. 이 metric은 목표 값과 비교되어 위와 같은 비율을 생성한다. autoscaling/v2 API 버전에서는, 비교가 이루어지기 전에 해당 값을 po의 개수로 선택적으로 나눌 수 있다.

hpa를 사용하는 일반적인 방법은 aggregated API(metrics.k8s.io, custom.metrics.k8s.io, external.metrics.k8s.io)로부터 metric을 가져오도록 설정하는 것이다. metrics.k8s.io API는 보통 metric server라는 add-on에 의해 제공된다.

[Support for metrics APIs](https://v1-24.docs.kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/#support-for-metrics-apis)에서 각 API에 대해 설명한다.

hpa controller는 scaling을 지원하는 workload resource에 접근한다. 이러한 resource는 scale이라는 subresource를 가지고 있으며, 이 subresource는 replica 수를 동적으로 설정하고 각각의 현재 상태를 확인할 수 있는 인터페이스다.

### Algorighm details
기본적으로 hpa controller는 현재 meric 값과 desired metric 값 사이의 비율을 기준으로 동작한다:

> desiredReplicas = ceil[currentReplicas * ( currentMetricValue / desiredMetricValue )]

예를 들어, 현재 metric 값이 200m이고 desired 값이 100m이면 200.0 / 100.0 == 2.0 수식에 의해 replica는 2배가 된다. 반대로 현재 값이  50m일 경우, 20.0 / 100.0 == 0.5 수식에 의해 replica는 1/2배가 된다. control plane은 비율이 1.0에 충분히 가까울 경우(기본적으로 오차는 0.1) scaling 동작을 수행하지 않는다.

.spec.metrics[].\*.target.averageValue 또는 .spec.metrics[].\*.target.targetAverageUtilization이 지정된 경우 currentMetricValue는 hpa의 scale target인 모든 po의 metric 평균 값을 통해 계산된다.

오차를 확인하고 최종 값을 결정하기 전에 control plane은 누락된 metric 여부, Ready po의 개수를 고려한다. [deletion timestamp가 지정된 po, failed pod는 버려진다.](https://github.com/kubernetes/kubernetes/blob/30c9f097ca4a26dab9085832e006f09cb2993dda/pkg/controller/podautoscaler/replica_calculator.go#L365)

[만약 일부 po에 대해 metric이 없을 경우 나중을 위해 별도로 남겨둔다.](https://github.com/kubernetes/kubernetes/blob/30c9f097ca4a26dab9085832e006f09cb2993dda/pkg/controller/podautoscaler/replica_calculator.go#L375) 누락된 metric이 있는 po는 최종 scale 크기를 결정할 때 사용된다.

CPU에 대해 scaling을 수행할 떄 ready 상태가 아닌(초기화 단계이거나 unhealthy) po 또는 마지막 metric 값이 ready 상태가 되기 전의 값이라면 해당 po 역시 별도로 남겨둔다.

기술적 문제로 hpa controller는 특정 cpu metric을 별도로 남겨두기를 결정할 때 po가 가장 처음으로 ready된 시간을 정확하게 알 수 없다. 대신 설정 가능한 시간(--horizontal-pod-autoscaler-initial-readiness-delay flag로 기본 값은 30초) 내에 unready 상태거나 unready로 상태가 변경되었을 때 "not yet ready" po로 고려한다. po가 ready 상태가 된 이후, 설정 가능한 시간(--horizontal-pod-autoscaler-cpu-initialization-period flag로 기본 값은 5분) 동안 추가적인 상태 변화는 po의 최초 준비로 간주된다. 최초 po가 ready가 된 후 해당 시간 동안의 cpu 샘플은 무시된다.

- --horizontal-pod-autoscaler-initial-readiness-delay: hpa controller가 새롭게 생성된 po를 ready 상태로 인지하기까지 기다리는 시간(pod startTime으로 부터)
- --horizontal-pod-autoscaler-cpu-initialization-period: 새롭게 생성된 po의 CPU 사용률을 계산하기까지 대기하는 시간. 새롭게 생성된 po의 경우 초기화 과정에서 많은 CPU를 사용할 수 있다.

그런 다음 별도로 남겨진 po와 버려진 po를 제외한 모든 po에 대해 currentMetricValue / desiredMetricValue 수식을 사용해 scale 비율을 게산한다.

누락된 metric이 있는 경우 control plane은 해당 po가 scale down 시에는 100% 사용, scale up 시에는 0% 사용한다고 보수적으로 예측한다.

또한 not-yet-ready po가 존재하고 누락된 metric 또는 not-yet-ready po를 고려하지 않고 워크로드가 scal up 됐을 경우 control plane은 not-yet-ready po가 scale up 시에는 0% 사용한다고 보수적으로 예측한다.

controller는 not-yet-ready po, metric이 누락된 po를 고려한 후 사용량 비율을 다시 계산한다. 다시 계산한 비율이 scale 방향과 반대거나 오차 범위 이내일 경우 scale 작업을 수행하지 않는다.

not-yet-ready po, 누락 metric에 대한 새로운 비율 값을 사용하더라도 hpa status에는 원래 값이 노출됨을 주의해야 한다.

관련된 k8s 내 hpa controller의 소스코드는 [github](https://github.com/kubernetes/kubernetes/blob/30c9f097ca4a26dab9085832e006f09cb2993dda/pkg/controller/podautoscaler/replica_calculator.go#L361)을 참고한다. 아래는 일부 소스코드다:

``` go
func groupPods(pods []*v1.Pod, metrics metricsclient.PodMetricsInfo, resource v1.ResourceName, cpuInitializationPeriod, delayOfInitialReadinessStatus time.Duration) (readyPodCount int, ignoredPods sets.String, missingPods sets.String) {
	missingPods = sets.NewString()
	ignoredPods = sets.NewString()
	for _, pod := range pods {
		if pod.DeletionTimestamp != nil || pod.Status.Phase == v1.PodFailed {
			continue
		}
		// Pending pods are ignored.
		if pod.Status.Phase == v1.PodPending {
			ignoredPods.Insert(pod.Name)
			continue
		}
		// Pods missing metrics.
		metric, found := metrics[pod.Name]
		if !found {
			missingPods.Insert(pod.Name)
			continue
		}
		// Unready pods are ignored.
		if resource == v1.ResourceCPU {
			var ignorePod bool
			_, condition := podutil.GetPodCondition(&pod.Status, v1.PodReady)
			if condition == nil || pod.Status.StartTime == nil {
				ignorePod = true
			} else {
				// Pod still within possible initialisation period.
				if pod.Status.StartTime.Add(cpuInitializationPeriod).After(time.Now()) {
					// Ignore sample if pod is unready or one window of metric wasn't collected since last state transition.
					ignorePod = condition.Status == v1.ConditionFalse || metric.Timestamp.Before(condition.LastTransitionTime.Time.Add(metric.Window))
				} else {
					// Ignore metric if pod is unready and it has never been ready.
					ignorePod = condition.Status == v1.ConditionFalse && pod.Status.StartTime.Add(delayOfInitialReadinessStatus).After(condition.LastTransitionTime.Time)
				}
			}
			if ignorePod {
				ignoredPods.Insert(pod.Name)
				continue
			}
		}
		readyPodCount++
	}
	return
}
```

hpa에 여러 metric이 명시된 경우 각 metric에 대해 계산이 이루어지며 가장 큰 desired replica 값이 선택된다. 여러 metric 중 desired replica 값이 계산될 수 없거나(예를 들어 metrics API로 부터 metric을 가져오는 데 에러가 발생) scale down이 제안되면 scaling은 건너뛴다. 물론 여러 metric 중 한개 metric이라도 desiredReplicas 값이 현재 값보다 클 경우 scale 동작을 수행할 것이다.

최종적으로 hpa이 target을 scale하기 직전에 scale recommendation이 기록된다. controller는 설정 가능한 window 내의 모든 recommendation 중 가장 높은 recommendation을 선택한다. 이 값은 --horizontal-pod-autoscaler-downscale-stabilization flag(기본값 5분)를 사용해 설정할 수 있다. 즉, 빠르게 변하는 metric 값에 대비해 scaledown이 점차적으로 부드럽게 수행될 수 있도록 한다.

## API Object
hpa는 k8s autoscaling API 그룹의 API resource다. 현재 stable 버전은 autoscaling/v2 API 버전이며, 메모리와 custom metric에 대한 scaling을 지원한다. autoscaling/v2에서 추가된 새로운 필드는 autoscaling/v1를 이용할 때에는 annotation으로 보존된다.

## Stability of workload scale
hpa를 사용해 replica 크기를 관리할 때 측정하는 metric의 동적 특성 때문에 replica 수가 자주 바뀔 수 있다. 이는 종종 thrashing 또는 flapping이라고 불린다. 이는 사이버네틱스 분야의 hysteresis 개념과 비슷하다.

## Autoscaling during rolling update
k8s는 deploy에 대한 rolling update를 지원한다. 이 경우 deploy가 rs을 알아서 관리한다. deploy에 autoscaling을 설정하려면, 각 deploy에 대한 hpa를 생성한다. hpa는 deploy의 replicas 필드를 관리한다. deploy controller는 rs의 replicas 값을 적용하여 rollout 과정 중/이후에 적절한 숫자까지 늘어나도록 한다.

autoscale된 replica가 있는 sts의 rolling update를 수행하면 sts가 직접 po의 수를 관리한다(즉, rs와 같은 중간 resource가 없다).

## Support for resource metrics
모든 hpa 타켓 scale 대상에서 po의 리소스 사용량을 기준으로 scale할 수 있다. po의 명세를 정의할 때는 cpu 및 memory 와 같은 resource request을 지정해야 한다. 이는 리소스 사용률을 결정하는 데 사용되며 hpa controller에서 대상을 scaling하거나 축소하는 데 사용한다. 리소스 사용률 기반 스케일링을 사용하려면 다음과 같은 metric 소스를 지정해야 한다:

``` yaml
type: Resource
resource:
  name: cpu
  target:
    type: Utilization
    averageUtilization: 60
```

이 metric을 사용하면 hap controller는 scaling 대상에서 po의 평균 사용률을 60%로 유지한다. 사용률은 po의 request resource에 대한 현재 리소스 사용량 간의 비율이다.

**Note**: 모든 container의 리소스 사용량이 합산되므로 총 po 사용량이 개별 container 리소스 사용량을 정확하게 나타내지 못할 수 있다. 이로 인해 단일 container가 높은 사용량일 때 전체 po 사용량은 여전히 허용 가능한 한도 내에 있기 때문에 hpa가 scaling 되지 않는 상황이 발생할 수 있다.

### Container resource metrics
hpa API는 개별 container에 대한 리소스 사용을 추적할 수 있도록 container metrics도 지원한다. 이를 통해 특정 po에서 가장 중요한 container에 대한 scaling 임계값을 설정할 수 있다. 예를 들어 웹 애플리케이션과 로깅 사이드카 container가 있는 경우 사이드카 container를 무시하고 웹 애플리케이션 container의 리소스 사용량 기준으로 scaling을 설정할 수 있다.

대상 리소스를 수정해 다른 container 집합을 갖는 새 po를 사용하는 경우, 새로 추가된 container도 scaling 대상일 경우 HPA sepc을 수정해야 한다. 명시된 container가 없거나 일부 po에만 속할 경우 이러한 po는 무시되며 recommendation이 재계산된다.

``` yaml
type: ContainerResource
containerResource:
  name: cpu
  container: application
  target:
    type: Utilization
    averageUtilization: 60
```

위 예시에서 hpa controller는 모든 po의 applciation이름의 container의 평균 cpu 사용량 60%을 기준으로 scaling을 수행한다.

**Note**: hpa가 추적하는 container의 이름을 변경하는 경우, 변경 사항을 적용하는 동안 scaling을 유효하도록 유지하기 위한 특정 순서가 있다. container를 정의하는 리소스(예를 들어 deploy)를 업데이트하기 전에 hpa를 업데이트해 새 container와 기존 container 이름 모두 추적하도록 변경해야 한다. hpa는 업데이트 동안 scaling recommendation을 계산할 수 있다.

roll out이 완료되면 이전 container 이름을 hpa에서 삭제하면 된다.

## Scaling on custom metrics
autoscaling/v2 API를 사용해 custom metric(k8s 구성 요소가 아님)을 통한 hpa를 설정할 수 있다. hpa controller는 k8s API에서 이러한 custom metric을 쿼리한다.

## Scaling on multiple metrics
autoscaling/v2 API를 사용해 여러 metric을 통한 hpa를 설정할 수 있다. hpa controller는 각 metric에 대해 계산을 수행한다. hpa는 각 metric 중 최대 scale recommendation을 취해 최종 값을 결정한다(설정한 전체 최대 값(.spec.maxReplicas)보다 크지 않을 경우).

## Support for metrics APIs
기본적으로 hpa controller는 API에서 metric을 탐색한다. 이러한 API에 접근하기 위해 클러스터 관리자는 아래 사항에 대해 보장해야 한다:

- API aggregation layer가 활성화
- 관련된 API가 등록되어 있어야 한다:
  - resource metric의 경우 일반적으로 metrics-server에서 제공하는 metrics.k8s.io API다. 이는 cluster add-on으로 제공된다.
  - custom metric의 경우 custom.metrics.k8s.io API다. metric 솔루션 공급 업체가 제공하는 "adapter" API server에서 제공된다. 사용 가능한 k8s metric adapter가 있는지 metric pipeline을 확인한다.
  - external metric의 경우 external.metrics.k8s.io API다. 이는 위에서 설명한 custom metric dapater에 의해 제공된다.

위와 같이 서로 다른 metric path의 디자인 목적은 [the HPA V2](https://git.k8s.io/design-proposals-archive/autoscaling/hpa-v2.md), [custom.metrics.k8s.io](https://git.k8s.io/design-proposals-archive/instrumentation/custom-metrics-api.md), [external.metrics.k8s.io](https://git.k8s.io/design-proposals-archive/instrumentation/external-metrics-api.md) 페이지를 참고한다.

## Configurable scaling behavior
v2 버전의 hpa API를 사용한다면 .spec.behavior 필드를 사용해 scale up 동작과 scale down 동작을 별도로 설정할 수 있다. 각각은 .spec.behavior 필드 아래의 scaleUp(.spec.behavior.scaleUp) / scaleDown(.spec.behavior.scaleDown)를 설정하여 지정할 수 있다.

.spec.behavior.*.stabilizationWindowSeconds 필드를 사용해 scaling 타겟의 replica 수 flapping을 방지할 수 있다. scaling 정책을 이용하여 scaling 시 replica 수 변화 속도를 조절할 수도 있다.

### Scaling policies
spec의 behavior 필드에 1개 이상의 scaling 정책이 명시될 수 있다. 여러 정책이 명시될 경우 변화의 크기가 가장 큰 정책이 기본적으로 선택된다. 아래 예시는 scale down에 대한 예시다:

``` yaml
behavior:
  scaleDown:
    policies:
    - type: Pods
      value: 4
      periodSeconds: 60
    - type: Percent
      value: 10
      periodSeconds: 60
```

periodSeconds 정책이 유지되어야 하는 시간을 의미한다. 즉, 첫 번째 정책의 경우 1분 내에 최대 4개의 po를 scale down한다. 두 번째 정책의 경우 1분 내에 최대 10%의 po를 scale down한다.

기본적으로 변화의 크기가 가장 큰 정책이 선택되므로 po가 40개 이상일 경우에만 두 번째 정책이 선택된다. 40개 이하일 경우 첫 번째 정책이 적용된다. 예를 들어 80개의 replica가 있는 상황에서 10개의 replica까지 scale down이 필요할 경우 첫 번째 단계에서는 8개의 replica가 줄어든다. 두 번째 단계에서는 72개의 replica 중 10%인 7.2의 올림 8개가 줄어든다. autoscaler controller의 각 루프마다 replica의 개수에 따라 po 변화량은 다시 계산된다. replica 개수가 40개 이하일 경우부터는 첫 번째 정책이 적용되어 한 번에 최대 4개의 replica가 줄어든다.

.spec.behavior.*.selectPolicy를 통해 정책 선택 방법을 변경할 수 있다. Min으로 설정할 경우 replica의 변화가 가장 작은 정책이 선택된다. Disabled로 설정할 경우 scaling을 비활성화 한다.

### Stabilization window
.spec.behavior.*.stabilizationWindowSeconds는 scaling에 사용되는 metric이 지속적으로 변동될 때 replica에 대한 flapping을 방지하는 데 사용된다. autoscaing 알고리즘은 이 window를 사용해 desired state를 추론하고 workload sacle에 대한 원하지 않는 변경을 방지한다.

아래는 scale down 시 예시다:

``` yaml
behavior:
  scaleDown:
    stabilizationWindowSeconds: 300
```

계산된 metric에 의해 target이 scale down되어야하는 경우 알고리즘은 이전의 desired state를 확인한다. 즉, 명시된 기간 내 최대 값을 확인한다. 위 예시의 경우 이전 5분에 대한 desired state 값을 확인한다.

이를 통해 scaling 알고리즘이 po를 자주 제거한느 것을 방지할 수 있다.

### Default Behavior
사용자는 모든 필드에 대해 값을 지정할 필요가 없다. 설정이 필요한 값만 명시하면 된다. 사용자가 설정한 값은 hpa의 기본 값과 병합된다. 아래는 hpa의 기본 설정이다:

``` yaml
behavior:
  scaleDown:
    stabilizationWindowSeconds: 300
    policies:
    - type: Percent
      value: 100
      periodSeconds: 15
  scaleUp:
    stabilizationWindowSeconds: 0
    policies:
    - type: Percent
      value: 100
      periodSeconds: 15
    - type: Pods
      value: 4
      periodSeconds: 15
    selectPolicy: Max
```

### Example: change downscale stabilization window
아래는 downscale stabilization window를 1분으로 설정한 예시다:

``` yaml
behavior:
  scaleDown:
    stabilizationWindowSeconds: 60
```

### Example: limit scale down rate
1분 내에 hpa에 의해 scaling 되는 po를 10%로 제한하기 위해 아래와 같이 설정한디:

``` yaml
behavior:
  scaleDown:
    policies:
    - type: Percent
      value: 10
      periodSeconds: 60
```

1분 내 최대 5개의 po만 scaling되도록 하기 위해 아래와 같이 두 번쨰 policy를 추가하고 selectPolicy 필드를 Min으로 설정한다:

``` yaml
behavior:
  scaleDown:
    policies:
    - type: Percent
      value: 10
      periodSeconds: 60
    - type: Pods
      value: 5
      periodSeconds: 60
    selectPolicy: Min
```

### Example: disable scale down
selectPolicy 필드 값을 disabled로 설정하면 해당 방향에 대한 scale을 비활성화한다. 아래는 예시다:

``` yaml
behavior:
  scaleDown:
    selectPolicy: Disabled
```

## Support for HorizontalPodAutoscaler in kubectl
다른 API와 같이 hpa도 kubectl 명령어에서 지원한다. kubectl create, get, delete, describe 명령어를 사용할 수 있다.

게다가 kubectl autoscale 명령어도 존재한다. 예를 들어 kubectl autoscale rs foo --min=2 --max=5 --cpu-percent=80 명령어는 foo라는 이름을 갖는 rs에 대한 hpa를 생성한다.

## Implicit maintenance-mode deactivation
hpa resource를 변경하지 않고 타켓에 대한 hpa를 비활성화할 수 있다. 타겟 workload resource의 replica를 0으로 설정하고 hpa의 최소 replica가 0보다 크면 hpa는 타겟의 replica이 다시 조정되거나, hpa의 최소 replica 수를 조정할 때까지 타겟 resource에 대한 auto scaling을 중지한다.

### Migrating Deployments and StatefulSets to horizontal autoscaling
hpa가 활성화 됐을때, deploy와 sts의 .spec.replicas 값을 삭제하는 것을 권장한다. 왜냐하면 kbectl apply -f <FILE_NAME> 명령어를 사용해 workload resource를 적용할 때 k8s가 FILE_NAME의 .spec.replicas 값을 사용해 po를 scaling 하기 때문이다. 이는 의도하지 않은 것이며 hpa가 활성화 됐을 때 문제를 야기할 수도 있다.

.spec.replica를 삭제하면 이 필드의 기본값인 1이 사용됨을 인지해야 한다. 값을 업데이트하면 po 1개를 제외하고 나머지 po가 종료된다. 이후 deploy 애플리케이션은 정상적으로 동작하며 rolling update 설정도 의도한 대로 동작한다. 아래 2가지 방법을 사용해 이러한 동작을 피할 수 있다.

- Client Side Apply (this is the default)
  1. kubectl apply edit-last-applied deployment/<deployment_name>
  2. 편집기에서 .spec.replicas 필드를 삭제한다. 저장 및 편집기를 종료하면 kbectl이 이 업데이트 사항을 적용한다. 이 단계에서 po 숫자가 변경되지 않는다.
  3. manifest에서 .spec.replica를 삭제할 수 있다. 소스 코드 관리 도구를 사용할 경우 변경 사항을 추적할 수 있도록 변경 사항을 커밋하고 추가 필요 단계를 수행한다.
  4. kubectl apply -f deployment.yaml를 실행할 수 있다.
- Server Side Apply
  - When using the Server-Side Apply you can follow the transferring ownership guidelines, which cover this exact use case.