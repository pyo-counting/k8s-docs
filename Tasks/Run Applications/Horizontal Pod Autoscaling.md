k8s에서 hpa는 workload resource(deploy, sts, rs, rc)를 자동으로 업데이트하며, workload의 replica를 수요에 맞게 자동으로 scaling하는 것을 목표로 한다.

horizontal scaling은 부하 증가에 맞춰 po를 더 배포하는 것을 뜻한다. 이는 vertical scaling(k8s에서는 실행 중인 po에 더 많은 자원(메모리 또는 CPU 등)을 할당하는 것)과는 다르다.

부하가 줄어들고 po의 수가 설정한 최소 값보다 많을 경우, hpa는 workload resourc에게 scale down을 지시한다.

hpa는 scale 조절이 불가능한 object(예를 들어 ds)에는 적용할 수 없다.

hpa는 k8s API resource와 controller를 통해 구현됐다. hpa API resource는 controller의 행동을 결정한다. k8s control plane 내에서 실행되는 hpa controller는 평균 CPU 사용률, 평균 메모리 사용률, 또는 다른 custom metric 등의 관측된 metric을 목표에 맞추기 위해 workdload의 replica 크기를 주기적으로 조정한다.

## How does a HorizontalPodAutoscaler work?
![How does a HorizontalPodAutoscaler work?](https://d33wubrfki0l68.cloudfront.net/4fe1ef7265a93f5f564bd3fbb0269ebd10b73b4e/1775d/images/docs/horizontal-pod-autoscaler.svg)

k8s는 hpa를 간헐적으로(intermittently) 실행되는 컨트롤 루프 형태로 구현했다(지속적인 프로세스가 아니다). 실행 주기는 kube-controller-manager의 --horizontal-pod-autoscaler-sync-period 파라미터에 의해 설정된다(기본 주기는 15초).

각 주기마다, controller manager는 각 hpa 정의에 지정된 metric에 대해 리소스 사용률을 질의한다. controller manager는 `.spec.scaleTargetRef`에 정의된 타겟 resource를 찾고, 타겟 resource의 `.spec.selector` label을 보고 po를 선택하며, 리소스 metric API(po 단위 리소스 metric) 또는 custom metric API(그 외 모든 meric 용)로부터 metric을 수집한다.

- po 단위 리소스 metric(예를 들어 CPU)의 경우 controller는 hpa가 대상으로하는 각 po에 대한 리소스 metric API에서 metric을 가져온다. 목표 사용률 값이 설정되면 controller는 각 po의 container에 대한 해당 resource request의 백분율로 사용률 값을 계산한다. 목표 정수값이 설정된 경우 metric 값을 계산없이 직접 사용한다. 그리고 controller는 모든 대상 po에서 사용된 사용률의 평균 또는 정수 값(지정된 대상 유형에 따라 다름)을 가져와서 원하는 replica 값을 scale하는데 사용되는 비율을 계산한다.

  po의 container 중 일부에 적절한 resource request가 설정되지 않은 경우, po의 CPU 사용률은 정의되지 않으며, 따라서 hpa는 해당 metric에 대해 아무런 조치도 취하지 않는다. autoscaling 알고리즘의 작동 방식에 대한 자세한 내용은 아래에서 살펴본다.

- po 단위 사용자 custom metric의 경우, controller는 사용률 값이 아닌 정수 값을 사용한다는 점을 제외하고는 po 단위 리소스 metric과 유사하게 동작한다.

- 오브젝트 metric, 외부 metric의 경우, 문제의 오브젝트를 표현하는 단일 metric을 가져온다. 이 metric은 목표 값과 비교되어 위와 같은 비율을 생성한다. autoscaling/v2 API 버전에서는, 비교가 이루어지기 전에 해당 값을 po의 개수로 선택적으로 나눌 수 있다.

hpa를 사용하는 일반적인 방법은 aggregated API(metrics.k8s.io, custom.metrics.k8s.io, external.metrics.k8s.io)로부터 metric을 가져오도록 설정하는 것이다. metrics.k8s.io API는 보통 metric server라는 add-on에 의해 제공된다.

hpa controller는 scaling을 지원하는 workload resource에 접근한다. 이러한 resource는 scale이라는 하위 resource를 가짖고 있으며, 이 하위 resource는 replica 수를 동적으로 설정하고 각각의 현재 상태를 확인할 수 있는 인터페이스다.

### Algorighm details
가장 기본적인 관점에서 hpa controller는 desired metric 값과 현재 metric 값 사이의 비율로 작동한다.

desiredReplicas = ceil[currentReplicas * ( currentMetricValue / desiredMetricValue )]

예를 들어 현재 metric 값이 200m이고 desired 값이 100m인 경우 200.0 / 100.0 == 2.0이므로 replica 값은 두 배가 된다. 만약 현재 값이 50m 이면, 50.0 / 100.0 == 0.5 이므로 replica 값을 반으로 줄일 것이다. control plane은 비율이 1.0(전역적으로 구성 가능한 허용 오차 내, 기본 값 0.1)에 충분히 가깝다면 scaling을 건너 뛸 것이다.

targetAverageValue 또는 targetAverageUtilization가 설정되면, currentMetricValue는 hpa의 scale 대상인 모든 po에 주어진 metric의 평균을 취하여 계산된다.

허용치를 확인하고 최종 값을 결정하기 전에 control plane은 누락된 metric이 있는지, 그리고 몇 개의 po가 Ready인지도 고려한다. deletion timestamp가 설정된 모든 po(po에 deletion timestamp가 있으면 shutdown 또는 삭제 중임을 의미)는 무시되며, 모든 실패한 po도 무시된다.

특정 po에 metric이 누락된 경우 나중을 위해 처리를 미뤄두는데 이와 같이 누락된 metric이 있는 모든 po는 최종 scale 크기를 조정하는데 사용된다.

CPU를 기준으로 scale 시 po가 아직 Ready되지 않았거나(초기화 중이거나 unhealthy 상태) 또는 po의 최신 metric 값이 준비되기 전이라면 마찬가지로 해당 po는 나중에 처리된다.

기술적 제약으로 인해, hpa controller는 특정 CPU metric을 나중에 사용할지 말지 결정할 때 po가 준비되는 시작 시간을 정확하게 알 수 없다. 대신, po가 아직 준비되지 않았고 시작 이후 짧은 시간 내에 po가 준비되지 않은 상태로 전환된다면 해당 po를 "아직 준비되지 않음(not yet ready)"으로 간주한다. 이 값은 --horizontal-pod-autoscaler-initial-readiness-delay 플래그로 설정되며, 기본값은 30초 이다. 일단 po가 준비되고 시작된 후 구성 가능한 시간 이내이면, 준비를 위한 어떠한 전환이라도 이를 시작 시간으로 간주한다. 이 값은 --horizontal-pod-autoscaler-cpu-initialization-period 플래그로 설정되며 기본값은 5분이다.

currentMetricValue / desiredMetricValue 기본 스케일 비율은 나중에 사용하기로 되어 있거나 위에서 폐기되지 않은 남아있는 po를 사용하여 계산된다.

누락된 metric이 있는 경우, control plane은 po가 scale down의 경우 원하는 값의 100%를 소비하고 scale up의 경우 0%를 소비한다고 가정하여 평균을 보다 보수적으로 재계산한다. 이것은 잠재적인 scale의 크기를 약화시킨다.

또한, 아직-준비되지-않은 po가 있고, 누락된 metric이나 아직-준비되지-않은 po가 고려되지 않고 workload가 scale up 된 경우, controller는 아직-준비되지-않은 po가 원하는 metric의 0%를 소비한다고 보수적으로 가정하고 스케일 확장의 크기를 약화시킨다.

아직-준비되지-않은 po나 누락된 metric을 고려한 후에, controller가 사용률을 다시 계산한다. 새로 계산한 사용률이 스케일 방향을 바꾸거나, 허용 오차 내에 있으면, controller가 스케일링을 건너뛴다. 그렇지 않으면, 새로 계산한 사용률를 이용하여 po 수 변경 결정을 내린다.

평균 사용량에 대한 원래 값은 새로운 사용 비율이 사용되는 경우에도 아직-준비되지-않은 po 또는 누락된 metric에 대한 고려없이 HorizontalPodAutoscaler 상태를 통해 다시 보고된다.

hpa에 여러 metric이 지정된 경우, 이 계산은 각 metric에 대해 수행된 다음 원하는 레플리카 수 중 가장 큰 값이 선택된다. 이러한 metric 중 어떠한 것도 원하는 레플리카 수로 변환할 수 없는 경우(예 : metric API에서 metric을 가져오는 중 오류 발생) 스케일을 건너뛴다. 이는 하나 이상의 metric이 현재 값보다 높은 desiredReplicas 을 제공하는 경우 HPA가 여전히 확장할 수 있음을 의미한다.

마지막으로, hpa가 목표를 스케일하기 직전에 스케일 권장 사항이 기록된다. controller는 구성 가능한 창(window) 내에서 가장 높은 권장 사항을 선택하도록 해당 창 내의 모든 권장 사항을 고려한다. 이 값은 --horizontal-pod-autoscaler-downscale-stabilization 플래그를 사용하여 설정할 수 있고, 기본값은 5분이다. 즉, 스케일 다운이 점진적으로 발생하여 급격히 변동하는 metric 값의 영향을 완만하게 한다.