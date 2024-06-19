k8s에서는 사용자의 요청이 인가(authorization, 접근 권한 부여)되기 전에 사용자가 인증(authentication, 로그인)되어야 한다. 일부 상황에서는 anonymous request를 허용하기도 한다.

## Authorization verdicts
인가는 kube-apiserver 내에서 이루어진다. kube-apiserver는 요청에 포함된 모든 속성을 모든 정책 기준으로 평가하고 필요에 따라 외부 서비스를 참고하며 최종적으로는 요청을 허용/거절한다.

API 요청이 진행되기 위해서는 인가 메커니즘에 의해 허용돼야 한다. 즉, 기본적으로 접근은 거부된다.

> **Note**:  
> 특정 종류의 resource의 특정 필드에 따라 접근 제어 및 정책을 평가하는 것은 admission controller가 수행한다.
>
> k8s admission controller는 인가가 완료된 후 진행된다(즉, 인가가 허용된 후에만 진행된다).

여러 인가 모듈이 활성화 된 경우 각각이 순차적으로 평가된다. 특정 authorizor가 요청을 승인하거나 거부하면 다른 authorizor와 상관없이 즉시 결과가 반환된다. 반대로 모든 authorizor에 대해 no opinion일 경우 기본적으로 거부된다. 이에 대해 kube-apiserver는 HTTP 403 status code(Forbidden)을 응답한다.

## Request attributes used in authorization
k8s는 API 요청에 포함된 속성 중 아래 목록만 확인한다.
- **user**: 인증 단계에서 제공되는 `user` 문자열
- **group**: 인증된 사용자가 속한 그룹의 목록
- **extra**: 인증 단계에서 제공되는 추가 문자열 key, value map
- **API**: resource 요청인지 여부를 나타낸다.
- **Request path**: non-reousrce endpoint(예를 들어 /api, /healthz)
- **API request verb**: resource 요청에 사용되는 verb(get, list, create, update, patch, watch, delete, deletecollection)
- **HTTP request verb**: non-resource 요청에 사용되는 소문자 HTTP method(get, post, put, delete)
- **Resource**: (resource 요청일 경우에 해당) 접근 resource의 이름 또는 ID. get, update, delete verb를 사용하는 경우 반드시 resource 이름을 제공해야 한다.
- **Subresource**: (resource 요청일 경우에 해당) 접근 subresource
- **Namespace**: The namespace of the object that is being accessed (for namespaced resource requests only).
- **API group**: The API Group being accessed (for resource requests only). An empty string designates the core API group.

### Request verbs and authorization
#### Non-resource requests
`/api/v1/...`(core group), `/apis/<group>/<version>/...`(name group) endpoint가 아닌 요청은 non-resource 요청으로 간주되며 요청의 HTTP method 소문자를 request verb로 사용한다. 예를 들어 `/api`, `healthz`와 같은 endpoint에 대한 HTTP GET 요청에 대해 get을 verb로 사용한다.

#### Resource requests
resource API endpoint에 대한 request verb를 결정하기 위해 k8s는 HTTP verb를 확인하고 요청이 개별 resource 또는 resource collection에 적용되는지 여부를 고려한다. 아래는 HTTP method에 대한 request verb 매핑 테이블이다.
| HTTP verb | request verb                                                                                                                                                  |
|-----------|---------------------------------------------------------------------------------------------------------------------------------------------------------------|
| POST      | create                                                                                                                                                        |
| GET, HEAD | get (개별 resource일 경우), list (for collections, including full object content), watch (for watching an individual resource or collection of resources) |
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
- **AlwaysAllow**: 이 모드는 모든 요청을 허용한다. 테스트와 같이 인가가 필요할 경우에만 사용한다.
- **AlwaysDeny**: 모든 요청을 거부한다. 테스트와 같이 인가가 필요하지 않은 경우에만 사용한다.
- **ABAC**: Kubernetes ABAC mode defines an access control paradigm whereby access rights are granted to users through the use of policies which combine attributes together. The policies can use any type of attributes (user attributes, resource attributes, object, environment attributes, etc).
- **RBAC**: k8s rbac는 개별 사용자의 role에 기반해 접근을 제어하는 방법이다. k8s는 rbac.authorization.k8s.io API 그룹을 사용해 인가 결정을 내리며 k8s API를 사용해 권한 정책을 동적으로 설정할 수 있도록 한다.
- **Node**: kubelet(사용자가 아닌)에 권한을 부여하는 특수한 목적의 authorization mode다. kubelet이 node authorizor에 의해 권한을 부여받기 위해 `system:nodes` 그룹, `ystem:node:<nodeName>` 이름을 가져야 한다.
- **Webhook**: Kubernetes webhook mode for authorization makes a synchronous HTTP callout, blocking the request until the remote HTTP service responds to the query.You can write your own software to handle the callout, or use solutions from the ecosystem.

> **Warning**:  
> AlwaysAllow mode는 모든 인가를 건너뛴다. 그렇기 때문에 신뢰할 수 없는 API 클라이언트 또는 workload를 실행하는 경우 사용하면 안된다.
> 
> AlwaysAllow을 활성화한다는 것은 다른 모든 authorizor가 no opinion을 반환할 경우 요청이 허용됨을 의미한다. 예를 들어 --authorization-mode=AlwaysAllow,RBAC flag는 거부에 대한 규칙이 없기 때문에 --authorization-mode=AlwaysAllow와 동일하다.

### Authorization mode configuration
kube-apiserver의 설정 파일 또는 명령어를 통해 authorizor chain을 구성할 수 있다.

두 방법을 같이 사용하면 안된다.

### Command line authorization mode configuration
- `--authorization-mode=ABAC` (Attribute-based access control mode)
- `--authorization-mode=RBAC` (Role-based access control mode)
- `--authorization-mode=Node` (Node authorizer)
- `--authorization-mode=Webhook` (Webhook authorization mode)
- `--authorization-mode=AlwaysAllow` (always allows requests; carries security risks)
- `--authorization-mode=AlwaysDeny` (always denies requests)

`--authorization-mode=Node,Webhook`와 같이 두 가지 mode를 설정할 수도 있다.

k8s는 나열한 순서에 따라 우선 순위를 결정하며 실행 순서를 결정한다.

`--authorization-mode`와 `--authorization-config`을 같이 사용할 수 없다.

### Configuring the API Server using an authorization config file

#### Authorization configuration and reloads

## Privilege escalation via workload creation or edits
ns에서 직접 또는 간접적인 [workload management](https://kubernetes.io/docs/concepts/architecture/controller/)를 허용하는 객체를 통해 po를 생성/편집할 수 있는 사용자는 해당 ns에서 권한 상승을 할 수도 있다. 권한 상승의 잠재적 경로에는 k8s [API extensions](https://kubernetes.io/docs/concepts/extend-kubernetes/#api-extensions)과 관련 컨트롤러가 포함된다.

> **Caution**  
> As a cluster administrator, use caution when granting access to create or edit workloads. Some details of how these can be misused are documented in escalation paths.

### Escalation paths
ns 내에서 임의의 po를 실행하도록 허용한다면 공격자 또는 신뢰할 수 없는 사용자는 해당 ns 내에서 추가 권한을 얻을 수 있는 다양한 방법이 있다.
- 해당 ns에 임의의 secret을 마운트
   - 다른 워크로드를 위한 secret에 접근할 수 있음
   - 보다 권한이 높은 sa의 토큰을 획득할 수 있음
- 해당 ns에서 임의의 sa 사용하기
   - 다른 워크로드로 k8s API 작업을 수행할 수 있음 (impersonation)
   - 해당 sa가 가진 모든 특권 작업을 수행할 수 있음
- 해당 ns에 다른 워크로드를 위해 의도된 cm을 마운트 또는 사용
   - 데이터베이스 호스트 이름과 같이 다른 워크로드를 위해 의도된 정보를 획득할 수 있음
- 해당 ns에 다른 워크로드를 위해 의도된 volume 마운트
   - 다른 워크로드를 위해 의도된 정보를 획득하고 변경할 수 있음

> **Caution**:  
> As a system administrator, you should be cautious when deploying CustomResourceDefinitions that let users make changes to the above areas. These may open privilege escalations paths. Consider the consequences of this kind of change when deciding on your authorization controls.

## Checking API access
kubectl 명령어는 auth can-i 하위 명령어를 제공해 API authorization layer에 쿼리할 수 있다. 이 명령어는 인가 모듈의 종류와 상관없이 현재 사용자가 지정된 작업을 수행할 수 있는지 여부를 알아내기 위해 SelfSubjectAccessReview API를 사용한다.
``` sh
kubectl auth can-i create deployments --namespace dev
```

관리자는 다른 사용자에 대해서도 해당 명령어를 사용할 수 있다.
``` sh
kubectl auth can-i list secrets --namespace dev --as dave
```

유사하게 dev ns의 dev-sa sa가 target ns의 po를 조회할 수 있는지 확인할 수 있다.
``` sh
kubectl auth can-i list pods \
    --namespace target \
    --as system:serviceaccount:dev:dev-sa
```

authorization.k8s.io API 그룹은 외부 서비스에 API server 인가를 노출하며 SelfSubjectAccessReview는 이 그룹에 포함된다. 해당 그룹에 포함되는 다른 resource는 다음과 같다:
- SubjectAccessReview:
- LocalSubjectAccessReview:
- SelfSubjectRulesReview:

``` sh
k api-resources | grep -iE "(authentication|authorization)" 
selfsubjectreviews                             authentication.k8s.io/v1          false        SelfSubjectReview
tokenreviews                                   authentication.k8s.io/v1          false        TokenReview
localsubjectaccessreviews                      authorization.k8s.io/v1           true         LocalSubjectAccessReview
selfsubjectaccessreviews                       authorization.k8s.io/v1           false        SelfSubjectAccessReview
selfsubjectrulesreviews                        authorization.k8s.io/v1           false        SelfSubjectRulesReview
subjectaccessreviews                           authorization.k8s.io/v1           false        SubjectAccessReview
clusterrolebindings                            rbac.authorization.k8s.io/v1      false        ClusterRoleBinding
clusterroles                                   rbac.authorization.k8s.io/v1      false        ClusterRole
rolebindings                                   rbac.authorization.k8s.io/v1      true         RoleBinding
roles                                          rbac.authorization.k8s.io/v1      true         Role
```

아래와 같이 k8s resource를 생성해 요청을 수행할 수 있으며 `.status`를 통해 결과를 확인할 수 있다.
``` sh
kubectl create -f - -o yaml << EOF
apiVersion: authorization.k8s.io/v1
kind: SelfSubjectAccessReview
spec:
  resourceAttributes:
    group: apps
    resource: deployments
    verb: create
    namespace: dev
EOF
```

예시 결과는 다음과 같다.
``` yaml
apiVersion: authorization.k8s.io/v1
kind: SelfSubjectAccessReview
metadata:
  creationTimestamp: null
spec:
  resourceAttributes:
    group: apps
    resource: deployments
    namespace: dev
    verb: create
status:
  allowed: true
  denied: false
```