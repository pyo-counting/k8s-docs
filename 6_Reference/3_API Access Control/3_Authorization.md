k8s에서는 사용자의 요청이 인가(authorization, 접근 권한 부여)되기 전에 사용자가 인증(authentication, 로그인)되어야 한다. 일부 상황에서는 anonymous request를 허용하기도 한다.

## Determine Whether a Request is Allowed or Denied
인가는 kube-apiserver 내에서 이루어진다. kube-apiserver는 요청에 포함된 모든 속성을 모든 정책을 기준으로 평가하고, 필요에 따라 외부 서비스를 참고해 최종적으로 요청을 허용하거나 거부한다.

API 요청이 진행되기 위해서는 인가 메커니즘에 의해 허용돼야 한다. 즉, 기본적으로 접근은 거부된다.

> **Note**:  
> 특정 종류의 resource의 특정 필드에 따라 접근 제어 및 정책을 평가하는 것은 admission controller가 수행한다.
>
> k8s admission controller는 인가가 완료된 후 진행된다(즉, 인가가 허용된 후에만 진행된다).

여러 인가 모듈이 활성화 된 경우 각각이 순차적으로 평가된다. 특정 authorizor가 요청을 승인하거나 거부하면 다른 authorizor와 상관없이 즉시 결과가 반환된다. 반대로 모든 authorizor에 대해 요청이 평가되지 않을 경우 거부된다. 이에 대해 kube-apiserver는 HTTP 403 status code(Forbidden)을 응답한다.

## Request attributes used in authorization
k8s는 API 요청에 포함된 속성 중 아래 목록만 확인한다.
- **user**: The user string provided during authentication.
- **group**: The list of group names to which the authenticated user belongs.
- **extra**: A map of arbitrary string keys to string values, provided by the authentication layer.
- **API**: Indicates whether the request is for an API resource.
- **Request path**: Path to miscellaneous non-resource endpoints like /api or /healthz.
- **API request verb**: API verbs like get, list, create, update, patch, watch, delete, and deletecollection are used for resource requests. To determine the request verb for a resource API endpoint, see request verbs and authorization.
- **HTTP request verb**: Lowercased HTTP methods like get, post, put, and delete are used for non-resource requests.
- **Resource**: The ID or name of the resource that is being accessed (for resource requests only) -- For resource requests using get, update, patch, and delete verbs, you must provide the resource name.
**Subresource**: The subresource that is being accessed (for resource requests only).
**Namespace**: The namespace of the object that is being accessed (for namespaced resource requests only).
**API group**: The API Group being accessed (for resource requests only). An empty string designates the core API group.

### Request verbs and authorization
#### Non-resource requests
`/api/v1/...`, `/apis/<group>/<version>/...` endpoint가 아닌 요청은 non-resource 요청으로 간주되며 요청의 HTTP method 소문자를 verb로 사용한다. 예를 들어 `/api`, `healthz`와 같은 endpoint에 대한 HTTP GET 요청에 대해 get을 verb로 사용한다.

#### Resource requests
resource API endpoint에 대한 request verb를 결정하기 위해 k8s는 HTTP verb를 매핑하고 요청이 개별 resource 또는 resource collection에 적용되는지 여부를 고려한다.
| HTTP verb | request verb                                                                                                                                                  |
|-----------|---------------------------------------------------------------------------------------------------------------------------------------------------------------|
| POST      | create                                                                                                                                                        |
| GET, HEAD | get (for individual resources), list (for collections, including full object content), watch (for watching an individual resource or collection of resources) |
| PUT       | update                                                                                                                                                        |
| PATCH     | patch                                                                                                                                                         |
| DELETE    | delete (for individual resources), deletecollection (for collections)                                                                                         |

> **Caution**:  
> get, list, watch verb는 resource의 전체 세부 정보를 반환할 수 있다. 반환된 데이터에 대한 접근 측면에서는 동등하다. 예를 들어 secret에 대한 list verb는 반환된 object의 데이터 속성도 노출한다.

k8s는 종종 특별한 verb를 사용해 부가적인 권한 인가를 확인한다. 예를 들어
- 인증에 대한 특별한 케이스
    - core API 그룹 내 users, groups, serviceaccounts resource, `authentication.k8s.io` API 그룹 내 userextras resource를 위한 impersonate verb
- [Authorization of CertificateSigningRequests](https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests/#authorization)
    - csr을 위한 approve verb, 기존 승인에 대한 수정을 위한 update verb
- RBAC
    - `rbac.authorization.k8s.io` API 그룹 내 role, clusterrole resource를 위한 bind, escalate verb

## Authorization context
k8s는 REST api 요청에 대한 일반적인 속성을 기대한다. 즉, k8s 인가는 k8s API 외의 다른 API를 처리할 수도 있는 기존 organization-wide 또는 cloud-provider-wide 접근 제어 시스템과 함께 동작할 수 있음을 의미한다.

## Authorization modes
kube-apiserver는 아래 인가 모드 중 하나를 사용해 요청을 인가한다:
- **AlwaysAllow**:
- **AlwaysDeny**:
- **ABAC**:
- **RBAC**:
- **Node**:
- **Webhook**:

## Checking API access
kubectl 명령어는 auth can-i 하위 명령어를 제공해 API authorization layer에 쿼리할 수 있다. 이 명령어는 인가 모듈의 종류와 상관없이 현재 사용자가 지정된 작업을 수행할 수 있는지 여부를 알아내기 위해 SelfSubjectAccessReview API를 사용한다.

``` bash
kubectl auth can-i create deployments --namespace dev
```

관리자는 다른 사용자에 대해서도 해당 명령어를 사용할 수 있다.

``` bash
kubectl auth can-i list secrets --namespace dev --as dave
```

authorization.k8s.io API 그룹은 외부 서비스에 API server 인가를 노출하며 SelfSubjectAccessReview는 이 그룹에 포함된다. 해당 그룹에 포함되는 다른 resource는 다음과 같다:

- SubjectAccessReview:
- LocalSubjectAccessReview:
- SelfSubjectRulesReview:

``` bash
kubectl api-resource -o wide
NAME                              SHORTNAMES   APIGROUP                       NAMESPACED   KIND                             VERBS
(...생략...)
localsubjectaccessreviews                      authorization.k8s.io           true         LocalSubjectAccessReview         [create]
selfsubjectaccessreviews                       authorization.k8s.io           false        SelfSubjectAccessReview          [create]
selfsubjectrulesreviews                        authorization.k8s.io           false        SelfSubjectRulesReview           [create]
subjectaccessreviews                           authorization.k8s.io           false        SubjectAccessReview              [create]
(...생략...)
```