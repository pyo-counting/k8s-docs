k8s에서 metrics API는 auto scaling과 유사한 케이스를 지원하기 위해 기본적인 metric 집합을 제공한다. 이 API는 no와 po의 CPU, 메모리 metric이 포함된 리소스 사용량를 제공한다. metrics API를 클러스터에 배포하면 k8s API의 클라이언트는 이 정보에 대해 쿼리할 수 있으며 쿼리 권한을 관리하기 위해 k8s의 접근 제어 메커니즘을 이용할 수 있다.

HorizontalPodAutoscaler(HPA), VerticalPodAutoscaler(VPA)는 사용자의 설정 사항을 만족하기 위해 replica와 리소스를 조정하는 데에 metrics API 정보를 이용한다.

kubectl top 명령어을 이용하여 리소스 metrics를 조회할 수도 있다.

![architecture of the resource metrics pipeline](/image/architecture of the resource metrics pipeline.png)

위 구조의 오른쪽부터 구성 요소는 다음과 같다:

- cAdvisor: kubelet에 포함된 container metric 수집, 집계, 노출을 담당하는 데몬
- kubelet: container 리소스 관리를 위한 no agent. 리소스 metric은 kubelet API 엔드포인트 /metrics/resource, /stats 를 통해 접근 가능하다.
- Summary API: /stats 엔드포인트를 통해 사용할 수 있는 no 별 요약된 정보를 탐색, 수집할 수 있도록 kubelet이 제공하는 API
- metrics-server: 각 kubelet으로부터 수집한 리소스 metric을 수집, 집계하는 클러스터 addon 구성 요소. API server는 hpa, vpa, kubectl top 명령어가 사용할 수 있도록 metrics API를 제공한다. metrics-server는 metrics API에 대한 reference implementation 중 하나이다.
- Metrics API: workload autoscaling에 사용되는 CPU, 메모리 정보 접근을 지원하는 k8s API. 이를 클러스터에서 사용하려면 metrics API를 제공하는 API 확장(extension) 서버가 필요하다.
