k8s에서는 사용자의 요청이 인가(authorization, 접근 권한 부여)되기 전에 사용자가 인증(authentication, 로그인)되어야 한다.

## Determine Whether a Request is Allowed or Denied
k8s는 API 서버를 이용하여 API 요청을 인가한다. 모든 정책과 비교하여 모든 요청 속성을 평가하고 요청을 허용하거나 거부한다. 요청이 계속 진행되기 위해 API 요청의 모든 부분이 일부 정책에 의해 반드시 허용되어야 한다. 이는 기본적으로 승인이 거부된다는 것을 의미한다.

(k8s는 API 서버를 사용하지만, 특정 object의 특정 필드에 의존하는 접근 제어 및 정책은 admission controll 모듈에 의해 처리된다.)

여러 개의 인가 모듈이 구성되면 각 모듈이 순서대로 호출된다. 인가 모듈이 요청을 승인하거나 거부할 경우, 그 결정은 즉시 반환되며 다른 인가 모듈은 호출되지 않는다. 모든 모듈에서 요청에 대해 인가되지 않으면 요청은 거부된다. 요청 거부는 HTTP status code 403을 반환한다.

## Review Your Request Attributes
k8s는 아래 API 요청 속성을 검토한다:

- **user**: The user string provided during authentication.
- **group**: The list of group names to which the authenticated user belongs.
- **extra**: A map of arbitrary string keys to string values, provided by the authentication layer.
- **API**: Indicates whether the request is for an API resource.
- **Request path**: Path to miscellaneous non-resource endpoints like /api or /healthz.
- **API request verb**: API verbs like get, list, create, update, patch, watch, delete, and deletecollection are used for resource requests. To determine the request verb for a resource API endpoint, see Determine the request verb.
- **HTTP request verb**: Lowercased HTTP methods like get, post, put, and delete are used for non-resource requests.
- **Resource**: The ID or name of the resource that is being accessed (for resource requests only) -- For resource requests using get, update, patch, and delete verbs, you must provide the resource name.
- **Subresource**: The subresource that is being accessed (for resource requests only).
- **Namespace**: The namespace of the object that is being accessed (for namespaced resource requests only).
- **API group**: The API Group being accessed (for resource requests only). An empty string designates the core API group.

## Determine the Request Verb
**Non-resource requests**: `/api/v1/...`, `/apis/<group>/<version>/...` endpoint가 아닌 요청은 non-resource 요청으로 간주되며 요청의 HTTP method 소문자를 요청 동사로 사용한다. 예를 들러 GET 요청은 get을 요청 동사로 사용한다.
**Resource requests**: resource API endpoint에 대한 요청 동사를 결정하기 위해, HTTP method와 해당 요청이 개별 resource 또는 resource 집합에 적용되는지 여부를 검사한다.

|HTTP method|요청 동사|
|------------|---------|
|POST|create|
|GET, HEAD|get(개별 resource), list(object 내용을 포함하는 집합), watch(개별 resource 또는 집합에 대한 watching)|
|PUT|update|
|PATCH|patch|
|DELETE|delete(개별 resource), deletecollection(집합)|

**Caution**: get, list, watch 동사는 resource의 상세 사항을 모두 반환한다. In terms of the returned data they are equivalent. For example, list on secrets will still reveal the data attributes of any returned resources.

k8s는 종종 특별한 요청 동사를 사용해 부가적인 권한 인가를 확인한다. 예를 들어:

- RBAC
    - rbac.authorization.k8s.io API 그룹의 roles, clusterroles resource에 대한 bind, escalate 요청 동사
- Authentication
    - impersonate verb on users, groups, and serviceaccounts in the core API group, and the userextras in the authentication.k8s.io API group.

## Authorization Modes
### Checking API Access
kubectl은 auth can-i 하위 명령어를 제공해 API authorization layer에 쿼리할 수 있다. 이 명령어는 인가 모듈의 종류와 상관없이 현재 사용자가 지정된 작업을 수행할 수 있는지 여부를 알아내기 위해 SelfSubjectAccessReview API를 사용한다.


## Using Flags for Your Authorization Module

## Privilege escalation via workload creation or edits
### Escalation paths