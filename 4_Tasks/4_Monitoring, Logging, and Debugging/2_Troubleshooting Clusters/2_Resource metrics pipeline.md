k8s에서 metrics API는 auto scaling과 유사한 케이스를 지원하기 위해 기본적인 metric 집합을 제공한다. 이 API는 no와 po의 CPU, 메모리 metric이 포함된 리소스 사용량를 제공한다. metrics API를 cluster에 배포하면 k8s API의 클라이언트는 이 정보에 대해 쿼리할 수 있으며 쿼리 권한을 관리하기 위해 k8s의 접근 제어 메커니즘을 이용할 수 있다.

HorizontalPodAutoscaler(hpa), VerticalPodAutoscaler(vpa)는 사용자의 요구 사항을 만족하기 위해 replica와 리소스를 조정하는 데에 metrics API 정보를 이용한다.

> **Note**:  
> metrics API와 metrics API를 활성화하는 metric pipeline은 hpa, vpa를 위한 최소한의 metric 정보만 제공한다. 더 완벽한 metric을 제공하기 위해 custom metrics API를 사용하는 두 번째 metric pipeline을 배포해 보완할 수 있다.

kubectl top 명령어을 이용하여 리소스 metrics를 조회할 수도 있다.

아래는 metric pipeline 아키텍처다.

![](../../../image/Resource%20Metrics%20Pipeline.png)

오른쪽부터 구성 요소는 다음과 같다:
- cAdvisor: kubelet에 포함된 container metric 수집, 집계, 노출을 담당하는 daemon
- kubelet: container 리소스 관리를 위한 no agent. 리소스 metric은 kubelet API endpoint `/metrics/resource`, `/stats` 를 통해 접근 가능하다.
- [node level resource metircs](https://kubernetes.io/docs/reference/instrumentation/node-metrics/): kubelet이 no의 요약된 통계를 제공하기 위한 API(`/metrics/resource`)
- metrics-server: 각 kubelet으로부터 수집한 리소스 metric을 수집, 집계하는 클러스터 addon 구성 요소. kube-apiserver는 hpa, vpa, kubectl top 명령어가 사용할 수 있도록 metrics API를 제공한다. metrics-server는 metrics API에 대한 reference implementation 중 하나이다.
- Metrics API: workload autoscaling에 사용되는 CPU, 메모리 정보 접근을 지원하는 k8s API. cluster에서 사용하기 위해 metrics API를 제공하는 API extension server가 필요하다.

> **Note**:  
> cAdvisor는 cgroups으로부터 metric을 가져오는 것을 지원하며 리눅스의 일반적인 container runtime은 이를 지원한다. 만약 다른 리소스 격리 메커니즘(예를 들어 가상화)을 사용하는 container runtime을 사용한다면 kubelet이 metric을 사용할 수 있기 위해서는 해당 container runtime이 [CRI Container Metrics](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-node/cri-container-stats.md)을 지원해야 한다.

## Metrics API
metrics-server는 metrics API를 구현한다. 이 API는 클러스터 내 no, po에 대한 CPU, memory 사용량을 제공한다. 주된 역할은 k8s autoscaler 구성 요소들에게 리소스 사용량 관련 metrics를 제공하는 것이다.

metrics API는 [k8s.io/metrics](https://github.com/kubernetes/metrics) repository에 정의되어 있다. `metrics.k8s.io` API를 위해 [API aggregation layer](https://kubernetes.io/docs/tasks/extend-kubernetes/configure-aggregation-layer/)를 활성화하고 [APIService](https://kubernetes.io/docs/reference/kubernetes-api/cluster-resources/api-service-v1/)를 활성화 해야 한다.

metrics API에 대해서는 [resource metrics API design](https://github.com/kubernetes/design-proposals-archive/blob/main/instrumentation/resource-metrics-api.md), [metrics-server repository](https://github.com/kubernetes-sigs/metrics-server), [resource metrics API]https://github.com/kubernetes/metrics#resource-metrics-api)를 참고한다.

> **Note**:  
> metrics API에 접근하기 위해 metrics-server 또는 대체 adapter를 배포해야 한다.

## Measuring resource usage
### CPU
### Memory

## Metrics Server
metrics-server는 kubelet으로부터 리소스 metric을 수집하고 이를 gpa, vpa가 활용할 수 있도록 kube-apiserver 내에서 metrics API를 통해 노출한다. kubectl top 명령을 사용하여 이 metric을 조회할 수도 있다.

metrics-server는 k8s API를 사용해 클러스터의 no, po를 추적한다. metrics-server는 각 no에 HTTP를 통해 질의하여 metric을 수집한다. metrics-server는 또한 po 메타데이터의 내부적 뷰를 작성하고, po health에 대한 캐시를 유지한다. 이렇게 캐시된 po health 정보는 metrics-server가 제공하는 extension API를 통해 이용할 수 있다.

예를 들어 hpa 쿼리의 경우, metrics-server는 deploy의 어떤 po가 label selector 조건을 만족하는지 판별해야 한다.

metrics-server는 각 no로부터 metric을 수집하기 위해 kubelet API를 호출한다. 사용 중인 metrics-server 버전에 따라 다음의 엔드포인트를 사용한다.
- v0.6.0 이상: metric 리소스 엔드포인트 `/metrics/resource`
- 이전 버전: summary API 엔드포인트 `/stats/summary`