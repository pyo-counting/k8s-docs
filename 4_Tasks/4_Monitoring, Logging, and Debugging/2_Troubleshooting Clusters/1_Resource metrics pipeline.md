k8s에서 metrics API는 auto scaling과 유사한 케이스를 지원하기 위해 기본적인 metric 집합을 제공한다. 이 API는 no와 po의 CPU, 메모리 metric이 포함된 리소스 사용량를 제공한다. metrics API를 클러스터에 배포하면 k8s API의 클라이언트는 이 정보에 대해 쿼리할 수 있으며 쿼리 권한을 관리하기 위해 k8s의 접근 제어 메커니즘을 이용할 수 있다.

HorizontalPodAutoscaler(hpa), VerticalPodAutoscaler(vpa)는 사용자의 설정 사항을 만족하기 위해 replica와 리소스를 조정하는 데에 metrics API 정보를 이용한다.

kubectl top 명령어을 이용하여 리소스 metrics를 조회할 수도 있다.

![architecture](../../../image/architecture%20of%20the%20resource%20metrics%20pipeline.png)

위 구조의 오른쪽부터 구성 요소는 다음과 같다:

- cAdvisor: kubelet에 포함된 container metric 수집, 집계, 노출을 담당하는 데몬
- kubelet: container 리소스 관리를 위한 no agent. 리소스 metric은 kubelet API 엔드포인트 /metrics/resource, /stats 를 통해 접근 가능하다.
- Summary API: /stats 엔드포인트를 통해 사용할 수 있는 no 별 요약된 정보를 탐색, 수집할 수 있도록 kubelet이 제공하는 API
- metrics-server: 각 kubelet으로부터 수집한 리소스 metric을 수집, 집계하는 클러스터 addon 구성 요소. API server는 hpa, vpa, kubectl top 명령어가 사용할 수 있도록 metrics API를 제공한다. metrics-server는 metrics API에 대한 reference implementation 중 하나이다.
- Metrics API: workload autoscaling에 사용되는 CPU, 메모리 정보 접근을 지원하는 k8s API. 이를 클러스터에서 사용하려면 metrics API를 제공하는 API 확장(extension) 서버가 필요하다.

**Note**: cAdvisor는 cgroups으로부터 metric을 가져오는 것을 지원하며 리눅스의 일반적인 container runtime은 이를 지원한다. 만약 다른 리소스 격리 메커니즘(예를 들어 가상화)을 사용하는 container runtime을 사용한다면 kubelet이 metric을 사용할 수 있기 위해서는 해당 container runtime이 CRI container metrics을 지원해야 한다.

## Metrics API
metrics-server는 Metrics API를 구현한다. 이 API는 클러스터 내 no, po에 대한 CPU, memory 사용량을 제공한다. 주된 역할은 k8s autoscaler 구성 요소들에게 리소스 사용량 관련 metrics를 제공하는 것이다.

## Measuring resource usage

## Metrics Server
metrics-server는 kubelet으로부터 리소스 metric을 수집하고 이를 gpa, vpa가 활용할 수 있도록 k8s API server 내에서 Metrics API를 통해 노출한다. kubectl top 명령을 사용하여 이 metric을 조회할 수도 있다.

metrics-server는 k8s API를 사용해 클러스터의 no, po를 추적한다. metrics-server는 각 no에 HTTP를 통해 질의하여 metric을 수집한다. metrics-server는 또한 po 메타데이터의 내부적 뷰를 작성하고, po health에 대한 캐시를 유지한다. 이렇게 캐시된 파드 health 정보는 metrics-server가 제공하는 extension API를 통해 이용할 수 있다.

예를 들어 HPA 쿼에 대한 경우, metrics-server는 deploy의 어떤 po가 label selector 조건을 만족하는지 판별해야 한다.

metrics-server는 각 no로부터 metric을 수집하기 위해 kubelet API를 호출한다. 사용 중인 metrics-server 버전에 따라 다음의 엔드포인트를 사용한다.

- v0.6.0 이상: metric 리소스 엔드포인트 /metrics/resource
- 이전 버전: 요약 API 엔드포인트 /stats/summary

### Summary API source
Kubelet은 no, volume, po, container 수준의 통계를 수집하며 소비자(consumer)가 읽을 수 있도록 이 통계를 Summary API에 기록한다.

**Note**: metrics-server 0.6.x 버전부터 Summary API /stats/summary 엔드포인트가 /metrics/resource 엔드포인트로 대체될 것이다.