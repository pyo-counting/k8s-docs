k8s에서 hpa는 workload resource(deploy, sts 등)를 자동으로 업데이트하며, workload의 replica를 수요에 맞게 자동으로 scaling하는 것을 목표로 한다.

horizontal scaling은 부하 증가에 맞춰 po를 더 배포하는 것을 뜻한다. 이는 vertical scaling(k8s에서는 실행 중인 po에 더 많은 자원(메모리 또는 CPU 등)을 할당하는 것)과는 다르다.

부하가 줄어들고 po의 수가 설정한 최소 값보다 많을 경우, hpa는 workload resourc에게 scale down을 지시한다.

hpa는 scale 조절이 불가능한 object(예를 들어ds)에는 적용할 수 없다.

hpa는 k8s API resource와 controller를 통해 구현됐다. hpa API resource는 controller의 행동을 결정한다. k8s control plane 내에서 실행되는 hpa controller는 평균 CPU 사용률, 평균 메모리 사용률, 또는 다른 custom metric 등의 관측된 metric을 목표에 맞추기 위해 workdload의 replica 크기를 주기적으로 조정한다.

## How does a HorizontalPodAutoscaler work?
![How does a HorizontalPodAutoscaler work?](https://d33wubrfki0l68.cloudfront.net/4fe1ef7265a93f5f564bd3fbb0269ebd10b73b4e/1775d/images/docs/horizontal-pod-autoscaler.svg)

k8s는 hpa를 간헐적으로(intermittently) 실행되는 컨트롤 루프 형태로 구현했다(지속적인 프로세스가 아니다). 실행 주기는 kube-controller-manager의 --horizontal-pod-autoscaler-sync-period 파라미터에 의해 설정된다(기본 주기는 15초).

각 주기마다, controller manager는 각 hpa 정의에 지정된 metric에 대해 리소스 사용률을 질의한다. controller manager는 scaleTargetRef에 정의된 타겟 resource를 찾고, 타겟 resource의 `.spec.selector` label을 보고 po를 선택하며, 리소스 metric API(파드 단위 리소스 metric) 또는 custom metric API(그 외 모든 메트릭 용)로부터 metric을 수집한다.

- po 단위 리소스 metric(예를 들어 CPU)의 경우 controller는 hpa가 대상으로하는 각 po에 대한 리소스 metric API에서 metric을 가져온다. 그런 다음 목표 사용률 값이 설정되면, controller는 각 po의 container에 대한 리로스 request 퍼센트 단위로 하여 사용률 값을 계산한다. 대상 원시 값이 설정된 경우 원시 메트릭 값이 직접 사용된다. 그리고, 컨트롤러는 모든 대상 파드에서 사용된 사용률의 평균 또는 원시 값(지정된 대상 유형에 따라 다름)을 가져와서 원하는 레플리카의 개수를 스케일하는데 사용되는 비율을 생성한다.

파드의 컨테이너 중 일부에 적절한 리소스 요청이 설정되지 않은 경우, 파드의 CPU 사용률은 정의되지 않으며, 따라서 오토스케일러는 해당 메트릭에 대해 아무런 조치도 취하지 않는다. 오토스케일링 알고리즘의 작동 방식에 대한 자세한 내용은 아래 알고리즘 세부 정보 섹션을 참조하기 바란다.

파드 단위 사용자 정의 메트릭의 경우, 컨트롤러는 사용률 값이 아닌 원시 값을 사용한다는 점을 제외하고는 파드 단위 리소스 메트릭과 유사하게 작동한다.

오브젝트 메트릭 및 외부 메트릭의 경우, 문제의 오브젝트를 표현하는 단일 메트릭을 가져온다. 이 메트릭은 목표 값과 비교되어 위와 같은 비율을 생성한다. autoscaling/v2 API 버전에서는, 비교가 이루어지기 전에 해당 값을 파드의 개수로 선택적으로 나눌 수 있다.