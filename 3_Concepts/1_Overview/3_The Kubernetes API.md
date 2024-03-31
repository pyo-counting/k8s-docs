k8s control plane의 핵심은 kube-apiserver다. kube-apiserver는 HTTP API를 노출하여 최종 사용자, cluster의 다양한 부분, 외부 구성 요소가 서로 통신할 수 있도록한다.

k8s API를 사용하면 k8s에서 API object의 state를 조회하고 수정할 수 있다(예: po, ns, cm 등).

대부분의 작업은 kubectl, kubeadm과 같은 CLI를 통해 수행된다. 이러한 도구들은 k8s API를 사용한다. 물론 REST call을 사용해 API에 직접 접근할 수도 있다. k8s는 k8s API를 사용하여 애플리케이션을 개발하려는 사람들을 위한 [client libraries](https://kubernetes.io/docs/reference/using-api/client-libraries/)를 제공한다.

각 k8s cluster는 cluster가 제공하는 API의 명세를 제공한다. k8s는 이러한 API 명세를 게시하기 위해 두 가지 메커니즘을 사용한다. 두 방법 모두 자동 상호 운용성을 활성화하는 데 유용하다. 예를 들어 kubectl는 자동완성 등의 기능을 활성화하기 위해 API 명세를 가져와 캐싱한. 지원되는 두 가지 메커니즘은 다음과 같다.
- Discovery API는 k8s API에 대한 정보를 제공한다. API 이름, resource, 버전, 동작. 이것은 k8s 특정 용어로, k8s OpenAPI와 별개의 API이기 때문에 사용 가능한 resource의 간단한 요약을 제공하며 리소스의 특정 스키마를 자세히 설명하지는 않는다. 리소스 스키마에 대한 참조는 OpenAPI 문서를 참조해야 한다.
- k8s OpenAPI 문서는 모든 k8s API 엔드포인트에 대한 (완전한) OpenAPI v2.0 및 3.0 스키마를 제공한다. OpenAPI v3는 API에 대한 더 포괄적이고 정확한 보기를 제공하기 때문에 OpenAPI에 액세스하는 용도로 선호됩니다. 이것은 사용 가능한 모든 API 경로뿐만 아니라 각 엔드포인트에서 모든 작업에 대해 소비되고 생성되는 모든 리소스를 포함한다. 또한 클러스터가 지원하는 모든 확장 구성 요소도 포함한다. 이 데이터는 완전한 명세이며 Discovery API보다 훨씬 크다.

## Discovery API
Aggregated discovery
Unaggregated discovery
OpenAPI interface definition
OpenAPI V2
OpenAPI V3
Protobuf serialization
Persistence
API Discovery
Aggregated Discovery
API groups and versioning
API changes
API Extension
What's next