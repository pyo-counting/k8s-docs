k8s control plane의 핵심은 kube-apiserver다. kube-apiserver는 최종 사용자, cluster의 여러 부분 및 외부 구성 요소가 서로 통신할 수 있도록 하는 HTTP API를 노출한다.

k8s API를 사용하면 k8s에서 API object의 state를 조회하고 수정할 수 있다(예: po, ns, cm 등).

대부분의 작업은 kubectl, kubeadm과 같은 CLI를 통해 수행된다. 이러한 도구들은 k8s API를 사용한다. 물론 REST call을 사용해 API에 직접 접근할 수도 있다. k8s는 k8s API를 사용하여 애플리케이션을 개발하려는 사람들을 위한 [client libraries](https://kubernetes.io/docs/reference/using-api/client-libraries/)를 제공한다.

각 k8s cluster는 API spec를 제공한다. k8s는 이러한 API spec를 게시하기 위해 두 가지 메커니즘을 사용한다. 두 방법 모두 자동 상호 운용성을 활성화하는 데 유용하다. 예를 들어 kubectl는 자동완성 등의 기능을 활성화하기 위해 API spec를 가져와 캐싱한다. 지원되는 두 가지 메커니즘은 다음과 같다.
- Discovery API: API 이름, resource, version, 동작과 같은 k8s API에 대한 정보를 제공한다. 이는 k8s OpenAPI와 별개의 API로 k8s 관련 용어다. 사용 가능한 resource에 대한 간략한 요약을 제공하기 위한 것이며 resource에 대한 특정 스키마를 자세하게 제공하지는 않는다. resource 스키마에 대한 참조는 OpenAPI 문서를 참조해야 한다.
- k8s OpenAPI Document: 전체 k8s API endpoint에 대한 [OpenAPI v2.0 and 3.0 schemas](https://www.openapis.org/)를 제공한다. OpenAPI v3는 API에 대한 보다 포괄적이고 정확한 뷰를 제공하므로 OpenAPI에 접근하는 데 선호되는 방법이다. 이는 사용 가능한 모든 API 경로와 모든 endpoint의 모든 작업에 대해 소비, 생성되는 모든 resource를 포함한다. 또한 cluster가 지원하는 모든 확장성 구성 요소도 포함합니다. 데이터는 완전한 spec이며 Discovery API의 데이터보다 훨씬 크다.

## Discovery API
k8s는 Discovery API를 통해서 지원하는 모든 group version, resource 목록을 게시한다. 각 resource에 대해 아래 정보를 포함한다.
- 이름
- cluster scope 또는 namespace scope에 대한 분류
- endpoint url과 지원하는 동사(verbs)
- 대체 이름(예를 들어, pod의 경우 po)
- group, version, kind

API는 aggregated 형태, unaggregated 형태 모두 제공한다. aggregated discovery는 두 개의 endpoint(`/api`, `/apis`)를 제공하는 반면, unaggregated discovery는 각 group version에 대해 별도의 각각의 endpoint를 제공한다.

aggregated discovery와 unaggregated discovery의 가장 근본적인 차이는 네트워크 효율성과 클라이언트(예: kubectl)가 cluster에서 지원하는 모든 resource를 파악하기 위해 몇 번의 HTTP 요청을 보내야 하는가에 있다.
| 구분                | 집계되지 않은 디스커버리 (Unaggregated)   | 집계된 디스커버리 (Aggregated)                  |
|---------------------|-------------------------------------------|-------------------------------------------------|
| HTTP 요청 횟수      | 여러 번 (1 + N번, N = API 그룹/버전의 수) | 단 2번 (/api 및 /apis)                          |
| 데이터 탐색 방식    | 트리 구조를 계층별로 하나씩 탐색          | 전체 API 목록을 한 번의 응답으로 수신           |
| 클라이언트 요구사항 | 특별한 헤더가 없는 일반 HTTP GET 요청     | 특정 Accept 헤더 필수 포함                      |
| 성능 (속도)         | 느림 (CRD가 많은 클러스터에서 지연 발생)  | 매우 빠름 (네트워크 왕복(Round-trip) 시간 제거) |
| 도입 시기           | k8s의 전통적인 기본 방식                  | v1.30부터 안정화(Stable) 및 기본 활성화         |

### Aggregated discovery
k8s는 aggregated discovery에 대한 안정적인 지원을 제공하며 cluster에서 지원하는 모든 resource를 두 개의 endpoint(`/api`, `/apis`)를 통해 게시한다. 이 endpoint를 요청하면 cluster에서 discovery 데이터를 가져오기 위해 보내는 요청 수를 크게 줄일 수 있다. aggregated discovery resource를 나타내는 Accept header(`Accept: application/json;v=v2;g=apidiscovery.k8s.io;as=APIGroupDiscoveryList`)를 사용해 해당 endpoint를 요청함으로써 데이터에 접근할 수 있다.

Accept header를 사용해 resource 타입을 명시하지 않으면 `/api`, `/apis` endpoint의 기본 응답은 unaggregated discovery에 대한 문서다.

k8s 내장 resource에 대한 discovery 문서는 [k8s Github repo](https://github.com/kubernetes/kubernetes/blob/release-1.35/api/discovery/aggregated_v2.json)에서 확인 가능하다. 쿼리할 k8s cluster가 없는 경우 해당 github 문서를 참조하면 된다.

해당 endpoint는 ETag와 protobuf encoding을 지원한다.

aggregated discovery는 정보를 수집하고 합치는(Aggregation) 무거운 작업을 클라이언트가 아닌 kube-apiserver 자체에서 처리하도록 하여 네트워크 병목 현상을 해결한다. 덕분에 `kubectl api-resources` 같은 명령어의 실행 속도나, k8s controller가 처음 시작될 때의 속도가 획기적으로 빨라진다. 예를 들어 새로운 cluster에 처음 kubectl 명령어를 입력했을 때 `~/.kube/cache/discovery/` 경로에 로컬 캐시를 생성하느라 잠깐 멈칫하는 현상을 겪을 수 있다. aggregated discovery는 바로 이 캐싱 과정을 최적화하기 위해 도입된 k8s의 아키텍처 개선 사항이다.

#### `/api` vs `/apis`
| 구분                 | /api (Core API)                                     | /apis (Named API Groups)                                   |
|----------------------|-----------------------------------------------------|------------------------------------------------------------|
| 개념                 | k8s의 가장 기본적이고 오래된 핵심 resource          | 기능별로 모듈화된 resource, CRD                            |
| API group 이름       | 없음 (내부적으로는 "" 빈 문자열로 취급)             | 명시적인 그룹 이름 존재 (예: apps, networking.k8s.io)      |
| YAML의 apiVersion    | v1                                                  | <group>/<version> (예: apps/v1, networking.k8s.io/v1)      |
| 주요 resource (Kind) | Pod, Node, Service, Namespace, ConfigMap, Secret 등 | Deployment, StatefulSet, Ingress, RoleBinding, 각종 CRD 등 |
| URL 구조             | /api/v1/<resource>                                  | /apis/<group>/<version>/<resource>                         |

- `/api`: k8s가 처음 만들어졌을 초기에는 모든 resource가 단일 경로 아래에 있었다.
    - 특징: k8s가 동작하기 위해 가장 필수적인 '뼈대' 역할을 하는 resource가 모여 있다.
    - API 호출 예시: cluster의 모든 po 목록을 가져오려면 GET /api/v1/pods를 호출한다. group 이름 없이 바로 version(v1)이 나온다.
    - 왜 분리하지 않았나? 하위 호환성을 매우 엄격하게 유지해야 하는 가장 기본적인 구성 요소들이기 때문에, 다른 확장 기능들과 섞이지 않도록 독립적인 경로에 남겨두었다.
- `/apis`: k8s가 발전하고 기능이 방대해지면서 모든 것을 `/api` 하나에 때려 넣는 것이 불가능해졌다. 뿐만 아니라 네트워킹, 스토리지, 워크로드 등 각 기능별로 버전을 따로 관리(예: alpha -> beta -> stable)하고, 사용자가 자신만의 resource를 추가할 수 있도록 구조 확장이 되었다.
    - 특징: Core API와 다르게 resource가 논리적인 '그룹(Group)'으로 묶여 있다.
    - API 호출 예시: deploy 목록을 가져오려면 GET /apis/apps/v1/deployments를 호출합니다. 여기서 apps가 API group 이름이다.

### Unaggregated discovery
unaggregated discovery는 하위 endpoint에 대한 discovery 문서를 게시한다.

cluster에서 제공하는 모든 group version에 대한 목록은 `/api`, `/apis` endpoint를 통해 게시된다. 아래는 예시다.
``` json
{
  "kind": "APIGroupList",
  "apiVersion": "v1",
  "groups": [
    {
      "name": "apiregistration.k8s.io",
      "versions": [
        {
          "groupVersion": "apiregistration.k8s.io/v1",
          "version": "v1"
        }
      ],
      "preferredVersion": {
        "groupVersion": "apiregistration.k8s.io/v1",
        "version": "v1"
      }
    },
    {
      "name": "apps",
      "versions": [
        {
          "groupVersion": "apps/v1",
          "version": "v1"
        }
      ],
      "preferredVersion": {
        "groupVersion": "apps/v1",
        "version": "v1"
      }
    },
    ...
}
```

특정 group version에서 제공되는 resource 목록에 대한 discovery 문서를 얻기 위해서는 `/apis/<group>/<version>`(예: `/apis/rbac.authorization.k8s.io/v1alpha1`) endpoint에 대한 요청을 추가적으로 수행하면된다.

unaggregated discovery를 사용해 API spec을 가져오는 경우 크롤링을 수행해야 한다.
1. 초기 요청: 루트 엔드포인트(`/api`, `/apis`)를 호출
2. "목차" 응답: kube-apiserver는 실제 resource가 아닌, 지원하는 API group과 version의 목록을 반환. (예: "우리 cluster는 apps/v1과 rbac.authorization.k8s.io/v1을 지원해.")
3. N+1 쿼리 문제: 2번의 목록을 바탕으로 모든 group version에 대해 각각 따로따로 HTTP 요청을 다시 보내야 한다. 예를 들어 apps/v1 그룹 안에 Deployment나 StatefulSet이 있다는 것을 알아내기 위해 GET /apis/apps/v1을 추가로 호출해야 한다.

## OpenAPI interface definition
OpenAPI spec에 대한 자세한 내용은 [OpenAPI documentation](https://www.openapis.org/)을 참고한다.

k8s는 OpenAPI v2.0, OpenAPI v3.0을 모두 제공한다. OpenAPI v3는 k8s resource에 대해 보다 포괄적인(손실 없는) 표현을 제공하므로 OpenAPI에 접근하는 데 선호되는 방법이다. OpenAPI 버전 2의 한계로 인해, 게시된 OpenAPI에서 default, nullable, oneOf를 포함하되 이에 국한되지 않는 특정 필드가 누락(drop)된다.

### OpenAPI V2
kube-apiserver는 `/openapi/v2` endpoint를 통해 집계된(aggregated) OpenAPI v2 spec을 제공한다. 다음과 같이 요청 header를 사용하여 응답 형식을 요청할 수 있다.
| Header          | Possible values                                            | Notes                                        |
|-----------------|------------------------------------------------------------|----------------------------------------------|
| Accept-Encoding | gzip                                                       | not supplying this header is also acceptable |
| Accept          | application/com.github.proto-openapi.spec.v2@v1.0+protobuf | mainly for intra-cluster use                 |
|                 | application/json                                           | default                                      |
|                 | *                                                          | serves application/json                      |

> **Warning**:  
> OpenAPI schema의 일부로 게시된 유효성 검사 규칙은 완전하지 않을 수 있으며 대개 완전하지 않다. 추가적인 유효성 검사는 kube-apiserver 내에서 발생한다. 정확하고 완전한 검증을 원한다면, `kubectl apply --dry-run=server`를 실행하면 적용 가능한 모든 유효성 검사를 실행한다(또한 admission 단계의 검사도 활성화한다).

### OpenAPI V3
k8s는 해당 API에 대한 설명을 OpenAPI v3로 게시하는 것을 지원한다.

사용 가능한 모든 group/version 목록을 볼 수 있도록 `/openapi/v3` discovery endpoint가 제공된다. 이 endpoint는 JSON만 반환한다. 이러한 group/version은 다음 형식으로 제공된다.
``` json
{
    "paths": {
        ...,
        "api/v1": {
            "serverRelativeURL": "/openapi/v3/api/v1?hash=CC0E9BFD992D8C59AEC98A1E2336F899E8318D3CF4C68944C3DEC640AF5AB52D864AC50DAA8D145B3494F75FA3CFF939FCBDDA431DAD3CA79738B297795818CF"
        },
        "apis/admissionregistration.k8s.io/v1": {
            "serverRelativeURL": "/openapi/v3/apis/admissionregistration.k8s.io/v1?hash=E19CC93A116982CE5422FC42B590A8AFAD92CDE9AE4D59B5CAAD568F083AD07946E6CB5817531680BCE6E215C16973CD39003B0425F3477CFD854E89A9DB6597"
        },
        ....
    }
}
```
serverRelativeURL은 클라이언트 측 캐싱을 향상시키기 위해 불변의(immutable) OpenAPI 설명을 가리킨다. kube-apiserver는 이 목적을 위해 적절한 HTTP 캐싱 헤더도 설정한다(`Expires`는 향후 1년으로, `Cache-Control`은 immutable로 설정됨). 더 이상 사용되지 않는 URL이 사용되면 kube-apiserver는 최신 URL로의 리디렉션을 반환한다.

kube-apiserver는 `/openapi/v3/apis/<group>/<version>?hash=<hash>` endpoint에서 k8s group version 별로 OpenAPI v3 spec을 게시한다.

허용되는 요청 헤더는 아래 표를 참조한다.
| Header          | Possible values                                            | Notes                                        |
|-----------------|------------------------------------------------------------|----------------------------------------------|
| Accept-Encoding | gzip                                                       | not supplying this header is also acceptable |
| Accept          | application/com.github.proto-openapi.spec.v3@v1.0+protobuf | mainly for intra-cluster use                 |
|                 | application/json                                           | default                                      |
|                 | *                                                          | serves application/json                      |

OpenAPI V3를 가져오기 위한 Golang 구현체는 k8s.io/client-go/openapi3 패키지에서 제공된다.

k8s 1.35는 OpenAPI v2.0 및 v3.0을 게시한다. 가까운 장래에 3.1을 지원할 계획은 없다.

### Protobuf serialization
k8s는 주로 cluster터 내부 통신을 위한 대안적인 protobuf 기반 직렬화 형식을 구현한다. 이 형식에 대한 자세한 내용은 [Kubernetes Protobuf serialization](https://git.k8s.io/design-proposals-archive/api-machinery/protobuf.md) 설계 제안서와 API 오브젝트를 정의하는 Go 패키지에 있는 각 스키마의 IDL(Interface Definition Language) 파일을 참조한다.

## Persistence
k8s는 object의 직렬화된 state를 etcd에 저장한다.

## API groups and versioning
필드를 제거하거나 리소스 표현 구조를 더 쉽게 재구성할 수 있도록, 쿠버네티스는 /api/v1 또는 /apis/rbac.authorization.k8s.io/v1alpha1과 같이 각각 다른 API 경로에서 여러 API 버전을 지원합니다.

API가 시스템 리소스 및 동작에 대해 명확하고 일관된 뷰를 제공하도록 보장하고, 수명이 다했거나(end-of-life) 실험적인 API에 대한 액세스 제어를 가능하게 하기 위해 버전 관리는 리소스나 필드 수준이 아닌 API 수준에서 수행됩니다.

API를 더 쉽게 발전시키고 확장하기 위해, 쿠버네티스는 활성화하거나 비활성화할 수 있는 API 그룹을 구현합니다.

API 리소스는 API 그룹, 리소스 종류(type), 네임스페이스(네임스페이스 범위 리소스의 경우) 및 이름으로 구분됩니다. API 서버는 API 버전 간의 변환을 투명하게 처리합니다. 즉, 모든 다양한 버전은 실제로 동일하게 영구 저장된 데이터(persisted data)의 표현일 뿐입니다. API 서버는 동일한 기본 데이터를 여러 API 버전을 통해 제공할 수 있습니다.

예를 들어, 동일한 리소스에 대해 v1과 v1beta1이라는 두 가지 API 버전이 있다고 가정해 보겠습니다. 처음에 해당 API의 v1beta1 버전을 사용하여 오브젝트를 생성한 경우, v1beta1 버전이 더 이상 사용되지 않고(deprecated) 제거될 때까지 v1beta1 또는 v1 API 버전을 사용하여 해당 오브젝트를 읽고, 업데이트하고, 삭제할 수 있습니다. 해당 시점이 지나면 v1 API를 사용하여 오브젝트에 계속 액세스하고 수정할 수 있습니다.

### API changes
Any system that is successful needs to grow and change as new use cases emerge or existing ones change. Therefore, Kubernetes has designed the Kubernetes API to continuously change and grow. The Kubernetes project aims to not break compatibility with existing clients, and to maintain that compatibility for a length of time so that other projects have an opportunity to adapt.

In general, new API resources and new resource fields can be added often and frequently. Elimination of resources or fields requires following the API deprecation policy.

Kubernetes makes a strong commitment to maintain compatibility for official Kubernetes APIs once they reach general availability (GA), typically at API version v1. Additionally, Kubernetes maintains compatibility with data persisted via beta API versions of official Kubernetes APIs, and ensures that data can be converted and accessed via GA API versions when the feature goes stable.

If you adopt a beta API version, you will need to transition to a subsequent beta or stable API version once the API graduates. The best time to do this is while the beta API is in its deprecation period, since objects are simultaneously accessible via both API versions. Once the beta API completes its deprecation period and is no longer served, the replacement API version must be used.

Note:
Although Kubernetes also aims to maintain compatibility for alpha APIs versions, in some circumstances this is not possible. If you use any alpha API versions, check the release notes for Kubernetes when upgrading your cluster, in case the API did change in incompatible ways that require deleting all existing alpha objects prior to upgrade.
Refer to API versions reference for more details on the API version level definitions.


## API Extension
k8s API는 두 가지 방법을 통해 확장할 수 있다.
1. [Custom resources](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)는 kube-apiserver가 resource API를 제공하는 방법을 선언적으로 정의할 수 있다.
2. [aggregation layer](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/apiserver-aggregation/)을 사용해 k8s API을 구현할 수 있다.
