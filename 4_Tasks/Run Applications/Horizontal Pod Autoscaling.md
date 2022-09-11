k8s에서 hpa는 workload resource(deploy, sts, rs, rc)를 자동으로 업데이트하며, workload의 replica를 수요에 맞게 자동으로 scaling하는 것을 목표로 한다.

horizontal scaling은 부하 증가에 맞춰 po를 더 배포하는 것을 뜻한다. 이는 vertical scaling(k8s에서는 실행 중인 po에 더 많은 자원(메모리 또는 CPU 등)을 할당하는 것)과는 다르다.

부하가 줄어들고 po의 수가 설정한 최소 값보다 많을 경우 hpa는 workload resourc에게 scale down을 지시한다.

hpa는 scale 조절이 불가능한 object(예를 들어 ds)에는 적용할 수 없다.

hpa는 k8s API resource와 controller를 통해 구현됐다. hpa API resource는 controller의 행동을 결정한다. k8s control plane 내에서 실행되는 hpa controller는 평균 CPU 사용률, 평균 메모리 사용률, 또는 다른 custom metric 등의 관측된 metric을 목표에 맞추기 위해 workdload의 replica 크기를 주기적으로 조정한다.

## How does a HorizontalPodAutoscaler work?
![How does a HorizontalPodAutoscaler work?](https://d33wubrfki0l68.cloudfront.net/4fe1ef7265a93f5f564bd3fbb0269ebd10b73b4e/1775d/images/docs/horizontal-pod-autoscaler.svg)

k8s는 hpa를 간헐적으로(intermittently) 실행되는 컨트롤 루프 형태로 구현했다(지속적인 프로세스가 아니다). 실행 주기는 kube-controller-manager의 --horizontal-pod-autoscaler-sync-period 파라미터에 의해 설정된다(기본 주기는 15초).

각 주기마다, controller manager는 각 hpa 정의에 지정된 metric에 대한 리소스 사용률을 질의한다. controller manager는 `.spec.scaleTargetRef`에 정의된 타겟 resource를 찾고, 타겟 resource의 `.spec.selector` label을 보고 po를 선택하며, 리소스 metric API(po 단위 리소스 metric) 또는 custom metric API(그 외 모든 meric 용)로부터 metric을 수집한다.

- po 단위 리소스 metric(예를 들어 CPU)의 경우 controller는 hpa가 대상으로하는 각 po에 대한 리소스 metric API에서 metric을 가져온다. 목표 사용률 값이 설정되면 controller는 각 po의 container에 대한 해당 resource request의 백분율로 사용률 값을 계산한다. 목표 정수값이 설정된 경우 metric 값을 계산없이 직접 사용한다. 그리고 controller는 모든 대상 po에서 사용된 사용률의 평균 또는 정수 값(지정된 대상 유형에 따라 다름)을 가져와서 원하는 replica 값을 scale하는데 사용되는 비율을 계산한다.

  po의 일부 container에 적절한 resource request가 설정되지 않은 경우 po의 CPU 사용률은 정의되지 않는다. 따라서 hpa는 해당 metric에 대해 아무런 조치도 취하지 않는다. autoscaling 알고리즘의 작동 방식에 대한 자세한 내용은 아래에서 살펴본다.

- po 단위 사용자 custom metric의 경우, controller는 사용률 값이 아닌 정수 값을 사용한다는 점을 제외하고는 po 단위 리소스 metric과 유사하게 동작한다.

- 오브젝트 metric, 외부 metric의 경우, 문제의 오브젝트를 표현하는 단일 metric을 가져온다. 이 metric은 목표 값과 비교되어 위와 같은 비율을 생성한다. autoscaling/v2 API 버전에서는, 비교가 이루어지기 전에 해당 값을 po의 개수로 선택적으로 나눌 수 있다.

hpa를 사용하는 일반적인 방법은 aggregated API(metrics.k8s.io, custom.metrics.k8s.io, external.metrics.k8s.io)로부터 metric을 가져오도록 설정하는 것이다. metrics.k8s.io API는 보통 metric server라는 add-on에 의해 제공된다.

hpa controller는 scaling을 지원하는 workload resource에 접근한다. 이러한 resource는 scale이라는 하위 resource를 가지고 있으며, 이 하위 resource는 replica 수를 동적으로 설정하고 각각의 현재 상태를 확인할 수 있는 인터페이스다.

### Algorighm details

## API Object
hpa는 k8s autoscaling API 그룹의 API resource다. 현재 stable 버전은 autoscaling/v2 API 버전이며, 메모리와 custom metric에 대한 scaling을 지원한다. autoscaling/v2에서 추가된 새로운 필드는 autoscaling/v1를 이용할 때에는 annotation 보존된다.

## Stability of workload scale
hpa를 사용해 replica 크기를 관리할 때 측정하는 metric의 동적 특성 때문에 replica 수가 자주 바뀔 수 있다. 이는 종종 thrashing 또는 flapping이라고 불린다. 이는 사이버네틱스 분야의 hysteresis 개념과 비슷하다.

## Autoscaling during rolling update
k8s는 deploy에 대한 rolling update를 지원한다. 이 때 deploy가 rs을 알아서 관리한다. deploy에 autoscaling을 설정하려면, 각 deploy에 대한 hpa를 생성한다. hpa는 deploy의 replicas 필드를 관리한다. deploy controller는 rs의 replicas 값을 적용하여 rollout 과정 중/이후에 적절한 숫자까지 늘어나도록 한다.

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

이 metric을 사용하면 hap controller는 scaling 대상에서 po의 평균 사용률을 60%로 유지한다. 사용률은 po의 요청 리소스에 대한 현재 리소스 사용량 간의 비율이다.

**Note**: 모든 container의 리소스 사용량이 합산되므로 총 po 사용량이 개별 container 리소스 사용량을 정확하게 나타내지 못할 수 있다. 이로 인해 단일 container가 높은 사용량일 때 전체 po 사용량은 여전히 허용 가능한 한도 내에 있기 때문에 hpa가 scaling 되지 않는 상황이 발생할 수 있다.

### Container resource metrics

## Scaling on custom metrics

## Scaling on multiple metrics

## Support for metrics APIs

## Configurable scaling behavior
v2 버전의 hpa API를 사용한다면 behavior 필드를 사용해 scale up 동작과 scale down 동작을 별도로 설정할 수 있다. 각각은 behavior 필드 아래의 scaleUp / scaleDown를 설정하여 지정할 수 있다.

stabilization window를 사용해 scaling 타겟의 replica 수 flapping을 방지할 수 있다. scaling 정책을 이용하여 scaling 시 replica 수 변화 속도를 조절할 수도 있다.

### Scaling policies

### Stabilization window

### Default Behavior

### Example: change downscale stabilization window

### Example: limit scale down rate

### Example: disable scale down

## Support for HorizontalPodAutoscaler in kubectl

## Implicit maintenance-mode deactivation
hpa resource를 변경하지 않고 비활성화할 수 있다. 타겟 workload resource의 replica를 0으로 설정하고 hpa의 최소 replica가 0보다 크면 hpa는 타겟의 replica를 다시 조정하거나, hpa의 최소 replica 수를 조정할 때까지 hpa는 타겟 resource에 대한 auto scaling을 중지한다.

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