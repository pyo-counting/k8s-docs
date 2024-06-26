k8s control plane의 핵심은 kube-apiserver다. kube-apiserver는 HTTP API를 노출하여 최종 사용자, cluster의 다양한 부분, 외부 구성 요소가 서로 통신할 수 있도록한다.

k8s API를 사용하면 k8s에서 API object의 state를 조회하고 수정할 수 있다(예: po, ns, cm 등).

대부분의 작업은 kubectl, kubeadm과 같은 CLI를 통해 수행된다. 이러한 도구들은 k8s API를 사용한다. 물론 REST call을 사용해 API에 직접 접근할 수도 있다. k8s는 k8s API를 사용하여 애플리케이션을 개발하려는 사람들을 위한 [client libraries](https://kubernetes.io/docs/reference/using-api/client-libraries/)를 제공한다.

각 k8s cluster는 API의 명세를 제공한다. k8s는 이러한 API 명세를 게시하기 위해 두 가지 메커니즘을 사용한다. 두 방법 모두 자동 상호 운용성을 활성화하는 데 유용하다. 예를 들어 kubectl는 자동완성 등의 기능을 활성화하기 위해 API 명세를 가져와 캐싱한다. 지원되는 두 가지 메커니즘은 다음과 같다.
- Discovery API는 k8s API에 대한 정보를 제공한다. API 이름, resource, 버전, 동작. 이것은 k8s에서 사용하는 용어로 k8s OpenAPI와 별개의 API이기 때문에 사용 가능한 resource의 간단한 요약을 제공하며 리소스의 특정 스키마를 자세히 설명하지는 않는다. 리소스 스키마에 대한 참조는 OpenAPI 문서를 참조해야 한다.
- The Kubernetes OpenAPI Document provides (full) OpenAPI v2.0 and 3.0 schemas for all Kubernetes API endpoints. The OpenAPI v3 is the preferred method for accessing OpenAPI as it provides a more comprehensive and accurate view of the API. It includes all the available API paths, as well as all resources consumed and produced for every operations on every endpoints. It also includes any extensibility components that a cluster supports. The data is a complete specification and is significantly larger than that from the Discovery API.

## Discovery API
### Aggregated discovery
### Unaggregated discovery
## OpenAPI interface definition
### OpenAPI V2
### OpenAPI V3
### Protobuf serialization
## Persistence
## API groups and versioning
### API changes

## API Extension
k8s API는 두 가지 방법을 통해 확장할 수 있다.
1. [Custom resources](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)는 kube-apiserver가 resource API를 제공하는 방법을 선언적으로 정의할 수 있다.
2. [aggregation layer](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/apiserver-aggregation/)을 사용해 k8s API을 구현할 수 있다.
