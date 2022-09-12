애플리케이션을 스케일하고 신뢰할 수 있는 서비스를 제공하기 위해 애플리케이션이 배포되었을 때 애플리케이션이 어떻게 동작하는지를 이해해야 한다. container, po, svc, 그리고 전체 클러스터의 특성을 검사하여 k8s 클러스터 내의 애플리케이션 성능을 검사할 수 있다. k8s는 각 레벨에서 애플리케이션의 리소스 사용량에 대한 상세 정보를 제공한다. 이 정보는 애플리케이션의 성능을 평가하고 병목 현상을 제거하여 전체 성능을 향상할 수 있게 해준다.

k8s에서 애플리케이션 모니터링은 단일 모니터링 솔루션에 의존하지 않는다. 신규 클러스터에서는 resource metrics 또는 full metrics 파이프라인으로 모니터링 통계를 수집할 수 있다.

## Resource metrics pipeline
resource metric 파이프라인은 hpa controller와 같은 클러스터 구성요소나 kubectl top 명령어와 관련있는 제한된 metric 집합을 제공한다. 이 metric은 lightweight, short-term, in memory metrics-server에 의해서 수집되며 metrics.k8s.io API를 통해 노출된다.

metrics-server는 클러스터 상의 모든 no를 발견하고 각 no의 kubelet에 CPU와 메모리 사용량을 질의한다. Kubelet은 k8s master와 no 간의 다리 역할을 하면서 머신에서 구동되는 po와 container를 관리한다. kubelet은 각각의 po를 해당하는 container에 매치시키고 container runtime interface를 통해 container runtime에서 개별 container의 사용량 통계를 가져온다. container를 구현하기 위해 리눅스 cgroup, 네임스페이스를 활용하는 container runtime을 사용하며 해당 container runtime이 사용 통계치를 퍼블리싱 하지 않는 경우 kubelet은 해당 통계치를 (cAdvisor의 코드를 사용하여) 직접 조회 할 수 있다. 이런 통계가 어떻게 도착하든 kubelet은 취합된 po 리소스 사용량 통계를 metric-server 리소스 Metrics API를 통해 노출한다. 이 API는 kubelet의 인증이 필요한 읽기 전용 포트의 /metrics/resource/v1beta1를 통해 제공한다.

## Full metrics pipeline
pull metric 파이프라인은 보다 풍부한 metric에 접근할 수 있도록 해준다. k8s는 hpa와 같은 메커니즘을 활용해서 이런 metric에 대한 반응으로 클러스터의 현재 상태를 기반으로 자동으로 스케일링하거나 클러스터를 조정할 수 있다. 모니터링 파이프라인은 kubelet에서 metric을 가져와서 k8s에 custom.metrics.k8s.io와 external.metrics.k8s.io API를 구현한 어댑터를 통해 노출한다.

CNCF 프로젝트인 prometheus는 기본적으로 k8s, no, prometheus 자체를 모니터링할 수 있다. CNCF 프로젝트가 아닌 완전한 metric 파이프라인 프로젝트는 k8s 문서의 범위가 아니다.