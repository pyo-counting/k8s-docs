k8s API는 HTTP를 통해 제공되는 resource 기반(RESTful) 프로그래밍 인터페이스다. 이 API는 표준 HTTP method(POST, PUT, PATCH, DELETE, GET)를 통해 주요 resource를 조회, 생성, 업데이트, 삭제하는 작업을 지원한다.

일부 resource의 경우 API는 세부적인 권한 부여를 가능하게 하는 추가 subresource를 포함한다(예를 들어 po의 세부 정보와 로그 조회를 위한 별도의 뷰). 이러한 subresource는 편의성이나 효율성을 위해 다양한 형태로 resource를 수락하고 제공할 수 있다.

Kubernetes supports efficient change notifications on resources via watches. Kubernetes also provides consistent list operations so that API clients can effectively cache, track, and synchronize the state of resources.

## Kubernetes API terminology
k8s는 일반적으로 API 개념을 설명하기 위해 일반적인 RESTful 용어를 사용한다.
- resource 타입은 URL에서 사용되는 이름이다(예: pods, namespaces, services).
- 모든 resource 타입은 kind라고 불리는 구체적인 표현(object schema)을 가지고 있습니다(예: Pod, Namespace, Service).
- resource 타입의 인스턴스 목록은 collection이라고 한다.
- resource 타입의 단일 인스턴스는 resource라고 하며 일반적으로 object를 나타낸다.
- 일부 resource 타입의 경우, API는 resource 아래의 URI 경로로 표현되는 하나 이상의 subresource를 포함한다.

대부분의 k8s API resource 타입은 object다. 이는 cluster에서 po나 ns와 같은 개념의 구체적인 인스턴스를 나타낸다. 소수의 API resource 타입은 가상 resource다. 이는 object보다는 object에 대한 작업을 나타내며 예를 들어 권한 확인(subjectaccessreviews resource)이나, or the eviction sub-resource of a Pod (used to trigger API-initiated eviction).

### Object names
API를 통해 생성할 수 있는 모든 object는 유일한 object 이름을 가지고 있어 동일한 요청을 여러 번 실행해도 결과가 변하지 않는 멱등성을 보장한다. 그러나 가상 resource 타입은 조회할 수 없거나 멱등성에 의존하지 않는 경우 유일한 이름을 가지지 않을 수 있다. ns 내에서는 주어진 kind의 object는 한 번에 하나의 이름만 가질 수 있다. 하지만 object를 삭제하면 동일한 이름으로 새 object를 만들 수 있다. 일부 object는 ns에 속하지 않는다(예: nods). 이러한 object의 이름은 cluster 전체에서 유일해야 한다.

### API verbs
거의 모든 object resource 타입은 표준 HTTP method(GET, POST, PUT, PATCH, DELETE)를 지원한다. k8s는 HTTP method와 구별하기 위해 종종 소문자로 표기되는 자체 method를 사용한다.

k8s는 resource [collection](https://kubernetes.io/docs/reference/using-api/api-concepts/#collections)을 반환하는 것을 list라고 하여, 단일 resource를 조회하는 get과 구분한다. HTTP GET 요청에 `?watch` query parameter를 포함하면 k8s는 이를 get이 아닌 watch라고 인식한다(자세한 내용은 [Efficient detection of changes](https://kubernetes.io/docs/reference/using-api/api-concepts/#efficient-detection-of-changes) 참고).

PUT 요청의 경우, k8s는 기존 object의 상태에 따라 이를 생성(create) 또는 업데이트(update)로 분류한다. update는 patch와 다르다.

## Resource URIs
모든 resource 타입은 cluster 범위(`/apis/GROUP/VERSION/*`) 또는 ns 범위(`/apis/GROUP/VERSION/namespaces/NAMESPACE/*`)로 구분된다. ns 범위의 resource 타입은 해당 ns가 삭제될 때 함께 삭제되며 해당 resource 타입에 대한 접근은 ns 범위의 권한 검사를 통해 제어된다.

참고: core resource는 `/apis` 대신 `/api`를 사용하며 GROUP path segment를 생략한다.

예시:
- `/api/v1/namespaces`
- `/api/v1/pods`
- `/api/v1/namespaces/my-namespace/pods`
- `/apis/apps/v1/deployments`
- `/apis/apps/v1/namespaces/my-namespace/deployments`
- `/apis/apps/v1/namespaces/my-namespace/deployments/my-deployment`

resource collection에 접근할 수도 있다(예: 모든 no 나열). 아래 path는 collection, resource를 조회하는 데 사용된다.
- cluster 범위의 resource:
    - `GET /apis/GROUP/VERSION/RESOURCETYPE`: 해당 resource 타입의 resource collection 반환
    - `GET /apis/GROUP/VERSION/RESOURCETYPE/NAME`: 해당 resource 타입의 NAME을 가진 resource 반환
- ns 범위의 resource:
    -` GET /apis/GROUP/VERSION/RESOURCETYPE`: 모든 ns에서 해당 resource 타입의 모든 인스턴스 collection 반환
    - `GET /apis/GROUP/VERSION/namespaces/NAMESPACE/RESOURCETYPE`: NAMESPACE에서 해당 resource 타입의 모든 인스턴스 collection 반환
    - `GET /apis/GROUP/VERSION/namespaces/NAMESPACE/RESOURCETYPE/NAME`: NAMESPACE에서 NAME을 가진 해당 resource 타입의 인스턴스 반환

ns는 cluster 범위의 resource 타입이므로, `GET /api/v1/namespaces`를 사용하여 모든 ns의 목록("collection")을 조회할 수 있으며, `GET /api/v1/namespaces/NAME`을 사용하여 특정 ns에 대한 세부 정보를 조회할 수 있다.
- cluster 범위의 subresource: `GET /apis/GROUP/VERSION/RESOURCETYPE/NAME/SUBRESOURCE`
- ns 범위의 subresource: `GET /apis/GROUP/VERSION/namespaces/NAMESPACE/RESOURCETYPE/NAME/SUBRESOURCE`

각 subresource에 대해 지원되는 method는 object에 따라 다르므로 자세한 내용은 [API reference](https://kubernetes.io/docs/reference/kubernetes-api/)를 참고한다. 여러 resource에 대한 subresource에 접근하는 것은 불가능하다.