rbac는 조직 내 개별 사용자의 역할에 따라 컴퓨터 또는 네트워크 resource에 대한 접근을 규제하는 방법이다.

rbac 인가는 `rbac.authorization.k8s.io` API 그룹을 사용하여 인가 결정을 내리므로 k8s API를 통해 정책을 동적으로 설정할 수 있다.

rbac를 활성화하려면 kube-apiserver의 `--authorization-mode` flag 목록에 `RBAC`를 포함한다. 예를 들어
``` sh
kube-apiserver --authorization-mode=Example,RBAC --other-options --more-options
```

## API objects
RBAC API는 Role, ClusterRole, RoleBinding ClusterRoleBinding 네 가지 k8s resource를 통해 구현된다. 다른 k8s resource와 마찬가지로 kubectl과 같은 도구를 사용해 다룰 수 있다.

> **Caution**:  
> 이러한 object들은 설계상으로 접근 제한을 수행한다. cluster에 직접 반영하면서 어떻게 동작하는지 이해하기 위해서는 [privilege escalation prevention and bootstrapping](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#privilege-escalation-prevention-and-bootstrapping)을 참고한다.

### Role and ClusterRole
RBAC Role과 ClusterRole은 권한 집합을 나타내는 규칙이 포함한다. 권한은 추가하는 방식으로 구현된다("거절"에 대한 규칙은 없다).

Role은 항상 특정 ns 내에서 권한을 설정한다; Role을 생성할 때 Role이 속한 ns를 설정해야 한다.

이와 반대로 ClusterRole은 non-namespaecd resource다. k8s object는 namespaced 또는 non-namespaced이기 때문에 이를 구분하기 위해 2개의 rule이 있는 것이다(둘 다 될수는 없다).

ClusterRole과 관련된 예시는 다음과 같다
1. namespaced resource에 대한 권한을 정의하고 각 ns 내에서 접근 권한을 부여한다.
2. namespaced resource에 대한 권한을 정의하고 모든 ns에 대한 접근 권한을 부여 받는다.
3. cluster-scoped resource에 대한 권한을 정의한다.

ns 내에서 역할을 정의할 경우 Role을 사용하고 클러스터에서 역할을 정의할 경우 ClusterRole을 사용한다.

#### Role example
아래는 po에 대한 read 접근을 허용하는 데 사용할 수 있는 Role 예시다.
``` yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""] # "" indicates the core API group
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```

#### ClusterRole example
ClusterRole을 사용해 Role과 동일한 권한을 부여할 수 있다. ClusterRole은 cluster-scoped로 아래 목록에 대한 접근 권한을 부여할 수도 있다:
- cluster-scoped resources (예를 들어 no)
- non-resource endpoints (예를 들어 /healthz)
- 모든 ns에 대한 namespaced resources(예를 들어 po). 예를 들어 kubectl get po -A 명령어를 사용하기 위해 ClusterRole을 사용할 수 있다.

아래는 특정 또는 모든 ns의 secret에 대한 read 접근을 허용하는 데 사용할 수 있는 ClusterRole 예시다.
``` yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  # "namespace" omitted since ClusterRoles are not namespaced
  name: secret-reader
rules:
- apiGroups: [""]
  #
  # at the HTTP level, the name of the resource for accessing Secret
  # objects is "secrets"
  resources: ["secrets"]
  verbs: ["get", "watch", "list"]
```

### RoleBinding and ClusterRoleBinding
RoleBinding은 Role에 정의된 권한을 사용자나 사용자 그룹, sa 등의 주체(subject)에게 부여한다. RoleBinding은 주체 목록과 부여된 Role에 대한 참조 정보를 갖는다. RoleBinding은 특정 ns 내에서 권한을 부여하는 반면, ClusterRoleBinding은 클러스터 전역에서 해당 권한을 부여한다.

RoleBinding은 동일한 ns 내의 모든 Role을 참조할 수 있다. 또는 RoleBinding은 ClusterRole을 참조함으로써 ClusterRole을 RoleBinding의 ns에 바인딩할 수도 있다. cluster 내 모든 ns에 ClusterRole을 바인딩하려면 ClusterRoleBinding을 사용한다.

#### RoleBinding examples
아래는 default ns에서 jane이라는 user에게 pod-reader Role을 부여하는 RoleBinding이다. 이를 통해 jane은 default ns 내 모든 po를 read할 수 있다.
```` yaml
apiVersion: rbac.authorization.k8s.io/v1
# This role binding allows "jane" to read pods in the "default" namespace.
# You need to already have a Role named "pod-reader" in that namespace.
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
# You can specify more than one "subject"
- kind: User
  name: jane # "name" is case sensitive
  apiGroup: rbac.authorization.k8s.io
roleRef:
  # "roleRef" specifies the binding to a Role / ClusterRole
  kind: Role #this must be Role or ClusterRole
  name: pod-reader # this must match the name of the Role or ClusterRole you wish to bind to
  apiGroup: rbac.authorization.k8s.io
````

RoleBinding은 ClusterRole을 참조해 해당 ClusterRole에 정의된 권한을 RoleBinding의 ns 내 resource에 부여할 수도 있다. 이런 종류의 참조를 사용하면 cluster 전체에 걸쳐 일련의 일반적인 Role을 정의한 다음 여러 ns에서 재사용할 수 있다.

예를 들어 아래 RoleBinding은 ClusterRole을 참조하지만 dave라는 사용자는 RoleBinding가 속한 development ns에서만 secret를 read할 수 있다.
``` yaml
apiVersion: rbac.authorization.k8s.io/v1
# This role binding allows "dave" to read secrets in the "development" namespace.
# You need to already have a ClusterRole named "secret-reader".
kind: RoleBinding
metadata:
  name: read-secrets
  #
  # The namespace of the RoleBinding determines where the permissions are granted.
  # This only grants permissions within the "development" namespace.
  namespace: development
subjects:
- kind: User
  name: dave # Name is case sensitive
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

#### ClusterRoleBinding example
cluster 전체에 대한 권한을 부여하기 위해 ClusterRoleBinding을 사용할 수 있다. 아래는 모든 ns 내 secret에 대한 read 권한을 manager라는 그룹에 부여하는 ClusterRoleBinding 예시다.
``` yaml
apiVersion: rbac.authorization.k8s.io/v1
# This cluster role binding allows anyone in the "manager" group to read secrets in any namespace.
kind: ClusterRoleBinding
metadata:
  name: read-secrets-global
subjects:
- kind: Group
  name: manager # Name is case sensitive
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

바인딩을 생성한 후에는 해당 바인딩이 참조하는 Role, ClusterRole을 변경할 수 없다. 바인딩의 `.roleRef`를 변경하려고 시도하면 유효성 검사 오류가 발생한다. 바인딩의 `.roleRef`를 변경하기 위해서는 다시 생성해야 한다.

이러한 제한이 있는 이유는 크게 두 가지다.
1. Making roleRef immutable allows granting someone update permission on an existing binding object, so that they can manage the list of subjects, without being able to change the role that is granted to those subjects.

2. A binding to a different role is a fundamentally different binding. Requiring a binding to be deleted/recreated in order to change the roleRef ensures the full list of subjects in the binding is intended to be granted the new role (as opposed to enabling or accidentally modifying only the roleRef without verifying all of the existing subjects should be given the new role's permissions).

`kubectl auth reconcile` 명령어는 RBAC object를 포함하는 manifest 파일을 생성하거나 업데이트하며 바인딩이 참조하는 role이 변경되는 경우 삭제와 재생성을 수행한다.

### Referring to resources
k8s API에서 대부분의 resource는 object 이름의 문자열 표현을 사용해 접근한다. 예를 들어 Pod는 `pods`로 표현된다. RBAC(Role-Based Access Control)는 해당 API endpoint의 URL에 나타나는 이름과 동일한 이름을 사용해 resource를 참조한다. 일부 k8s API는 subresource를 포함한다. 아래는 po의 subresource인 log를 요청하는 예시다.
```
GET /api/v1/namespaces/{namespace}/pods/{name}/log
```

아래 예시의 경우 pods는 po resource의 namespaced resource이며 log는 pods의 subresource다. subresource는 RBAC role에서 표현하기위해 resource와 subresource를 /로 구분해 표현한다. 어떤 subject가 pods를 읽고 각 po의 log subresource에 접근할 수 있도록 허용하기 위해 아래와 같이 작성한다.
``` yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-and-pod-logs-reader
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list"]
```

또한, 특정 요청에 대해 `.rules[*].resourceNames`을 통해 resource를 이름으로 참조할 수 있다. 이 경우 요청은 개별 resource 인스턴스로 제한된다. 다음은 subject가 my-configmap이라는 cm을 조회하거나 업데이트할 수 있도록 제한하는 예시다
``` yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: configmap-updater
rules:
- apiGroups: [""]
  #
  # at the HTTP level, the name of the resource for accessing ConfigMap
  # objects is "configmaps"
  resources: ["configmaps"]
  resourceNames: ["my-configmap"]
  verbs: ["update", "get"]
Note:
```

> **Note**:  
> `.rules[*].resourceNames`을 사용해 `create` 또는 `deletecollection` 요청을 제한할 수 없다. create 요청의 경우, 새로운 object의 이름이 권한 부여 시점에 모를 수도 있기 때문이다(아직 생성지 않은 이름). `list`, `watch` 요청을 `.rules[*].resourceNames`으로 제한하면, 클라이언트는 `kubectl get configmaps --field-selector=metadata.name=my-configmap`와 같이 이름도 명시해서 요청해야 한다.

각각의 `.rules[*].resources`, `.rules[*].apiGroups`, `.rules[*].verbs`를 참조하는 대신, 와일드카드 `*` 문자를 사용해 모든 object를 참조할 수 있다. `.rules[*].nonResourceURLs`의 경우, 와일드카드 *를 접미사 glob match로 사용할 수 있다. `.rules[*].resourceNames`가 비어 있으면 모든 것이 허용된다는 의미다. 아래는 example.com API group의 현재, 미래의 모든 resource에 대해 현재, 미래의 모든 작업을 수행할 수 있도록 허용하는 예시다. 이는 내장된 cluster-admin role과 유사하다.
``` yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: example.com-superuser # DO NOT USE THIS ROLE, IT IS JUST AN EXAMPLE
rules:
- apiGroups: ["example.com"]
  resources: ["*"]
  verbs: ["*"]
```

> **Caution**:  
> `.rules[*].resources`, `.rules[*].verbs`에 와일드카드를 사용하면 민감한 resource에 대해 과도한 접근 권한이 부여될 수 있다. 예를 들어 새로운 resource 타입이 추가되거나 새로운 subresource가 추가되거나 새로운 custom verb가 체크되는 경우 와일드카드는 자동으로 접근 권한을 부여하기 때문에 적절하지 않다. [principle of least privilege](https://kubernetes.io/docs/concepts/security/rbac-good-practices/#least-privilege)을 적용해 특정 resource와 verb를 사용함으로써 워크로드가 올바르게 작동하는 데 필요한 권한만 부여되도록 해야한다.

### Aggregated ClusterRoles
여러 ClusterRole을 하나의 집계된 ClusterRole을 생성할 수 있다. `.aggregationRule`에 label selector 정의해 참조할 ClusterRole을 선택할 수 있다. kube-controller-manager는 label selector에 매칭되는 ClusterRole을 참조해 `.rules` 필드를 자동으로 채운다.

> **Caution**:  
> control plane은 집계된 ClusterRole의 `.rules` 필드에 사용자가 명시한 값은 덮어쓴다. rule을 변경하거나 추가하기 위해서는 집계된 ClusterRole이 참조하는 ClusterRole을 변경해야 한다.

아래는 집계된 ClusterRole 예시다.
``` yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring
aggregationRule:
  clusterRoleSelectors:
  - matchLabels:
      rbac.example.com/aggregate-to-monitoring: "true"
rules: [] # The control plane automatically fills in the rules
```

집계된 ClusterRole의 label selector에 매칭되는 새로운 ClusterRole을 생성하면 집계된 ClusterRole에도 새로운 rule이 추가된다. 아래는 위 집계된 ClusterRole의 label selector에 매칭되는 새로운 ClusterRole 예시다.
``` yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring-endpoints
  labels:
    rbac.example.com/aggregate-to-monitoring: "true"
# When you create the "monitoring-endpoints" ClusterRole,
# the rules below will be added to the "monitoring" ClusterRole.
rules:
- apiGroups: [""]
  resources: ["services", "endpointslices", "pods"]
  verbs: ["get", "list", "watch"]
```
[default user-facing roles](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#user-facing-roles)는 ClusterRole 집계를 사용한다. 이를 통해 관리자는 crd 또는 kube-apiserver, aggregated API server에서 제공하는 cr와 같은 rule을 포함하해 기본 role을 확장할 수 있다.

아래 ClusterRole은 "admin"과 "edit" role이 CronTab이라는 cr을 관리할 수 있도록 하며, "view" role은 CronTab resource에 대해 read 작업만 수행할 수 있다. CronTab resource kube-apiserver URL에서 "crontabs"로 표시된다.
``` yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: aggregate-cron-tabs-edit
  labels:
    # Add these permissions to the "admin" and "edit" default roles.
    rbac.authorization.k8s.io/aggregate-to-admin: "true"
    rbac.authorization.k8s.io/aggregate-to-edit: "true"
rules:
- apiGroups: ["stable.example.com"]
  resources: ["crontabs"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: aggregate-cron-tabs-view
  labels:
    # Add these permissions to the "view" default role.
    rbac.authorization.k8s.io/aggregate-to-view: "true"
rules:
- apiGroups: ["stable.example.com"]
  resources: ["crontabs"]
  verbs: ["get", "list", "watch"]
```

#### Role examples
아래 예시들은 Role, ClusterRole의 `.rules` 필드만 나타낸다.

core API group의 "pods" resource에 대한 read를 허용한다.
``` yaml
rules:
- apiGroups: [""]
  #
  # at the HTTP level, the name of the resource for accessing Pod
  # objects is "pods"
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
```

"apps" API group의 "deployments" resource에 대한 read/write를 허용한다.
``` yaml
rules:
- apiGroups: ["apps"]
  #
  # at the HTTP level, the name of the resource for accessing Deployment
  # objects is "deployments"
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

core API group의 "pods" resource에 대한 read, "batch" API group의 "jobs" resource에 대한 read를 허용한다.
``` yaml
rules:
- apiGroups: [""]
  #
  # at the HTTP level, the name of the resource for accessing Pod
  # objects is "pods"
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["batch"]
  #
  # at the HTTP level, the name of the resource for accessing Job
  # objects is "jobs"
  resources: ["jobs"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

core API group의 "configmaps" resource 중 이름이 "my-config"에 대한 read를 허용한다.
``` yaml
rules:
- apiGroups: [""]
  #
  # at the HTTP level, the name of the resource for accessing ConfigMap
  # objects is "configmaps"
  resources: ["configmaps"]
  resourceNames: ["my-config"]
  verbs: ["get"]
```

core API group의 "nodes" resource에 대한 read를 허용한다.
``` yaml
rules:
- apiGroups: [""]
  #
  # at the HTTP level, the name of the resource for accessing Node
  # objects is "nodes"
  resources: ["nodes"]
  verbs: ["get", "list", "watch"]
```

non-resource endpoint인 "/healthz"와 하위 path에 대한 GET, POST 요청을 허용한다.
``` yaml
rules:
- nonResourceURLs: ["/healthz", "/healthz/*"] # '*' in a nonResourceURL is a suffix glob match
  verbs: ["get", "post"]
```

### Referring to subjects
RoleBinding 또는 ClusterRoleBinding은 Role 또는 CLusterRole을 주체(subject)와 연결한다. 주체는 그룹, 사용자, sa가 될 수 있다.

k8s는 사용자 이름을 문자열로 나타낸다. 이름은 "alice"와 같은 일반 이름, "bob@example.com"과 같은 이메일 형식의 이름 또는 문자열로 표현된 숫자 사용자 ID일 수 있다. 관리자는 authenticator module을 구성하여 원하는 형식으로 사용자 이름을 생성하도록 해야한다.

> **Caution**:  
> 접두사 `system:`은 k8s가 시스템 용도로 예약되어 있으므로 `system:`으로 시작하는 이름을 가진 사용자나 그룹이 생기지 않도록 주의해야한다. 이 특별한 접두사를 제외하고, RBAC authorization 시스템은 사용자 이름에 대해 형식을 제한하지 않는다.

k8s에서 authenticator module은 그룹 정보를 제공한다. 그룹은 사용자와 마찬가지로 문자열로 표현되며, 이 문자열도 형식 요구 사항이 없지만 `system:` 접두사는 예약되어 있다.

sa는 `system:serviceaccount:` 접두사가 붙은 이름을 가지며, `system:serviceaccounts:` 접두사가 붙은 이름을 가진 그룹에 속한다.

> **Note**:  
> - `system:serviceaccount`: sa 사용자 이름의 접두사
> - `system:serviceaccounts`: sa 그룹 이름의 접두사

#### RoleBinding examples
아래 예시들은 RoleBinding의 `.subjects` 필드만 나타낸다.

사용자 이름이 `alice@example.com`
``` yaml
subjects:
- kind: User
  name: "alice@example.com"
  apiGroup: rbac.authorization.k8s.io
```

그룹 이름이 `frontend-admins`
``` yaml
subjects:
- kind: Group
  name: "frontend-admins"
  apiGroup: rbac.authorization.k8s.io
```

"kube-system" ns에 속한 default sa
``` yaml
subjects:
- kind: ServiceAccount
  name: default
  namespace: kube-system
```

"qa" ns에 속한 모든 sa
``` yaml
subjects:
- kind: Group
  name: system:serviceaccounts:qa
  apiGroup: rbac.authorization.k8s.io
```

모든 ns에 속한 모든 sa
``` yaml
subjects:
- kind: Group
  name: system:serviceaccounts
  apiGroup: rbac.authorization.k8s.io
```

모든 authenticated 사용자
``` yaml
subjects:
- kind: Group
  name: system:authenticated
  apiGroup: rbac.authorization.k8s.io
```

모든 unauthenticated 사용자
``` yaml
subjects:
- kind: Group
  name: system:unauthenticated
  apiGroup: rbac.authorization.k8s.io
```

모든 사용자
``` yaml
subjects:
- kind: Group
  name: system:authenticated
  apiGroup: rbac.authorization.k8s.io
- kind: Group
  name: system:unauthenticated
  apiGroup: rbac.authorization.k8s.io
```

## Default roles and role bindings
kube-apiserver는 기본 ClusterRole, ClusterRoleBinding object를 생성한다. 대부분의 object는 `system:` 접두사 이름을 갖으며, 이는 해당 object가 control plane에 의해 관리됨을 의미한다. 모든 기본 ClusterRole,ClusterRoleBinding은 `kubernetes.io/bootstrapping=rbac-defaults` label을 갖고 있다.

> **Caution**:  
> `system:` 접두사가 붙은 ClusterRole, ClusterRoleBinding을 수정할 때는 주의해야 한다. 이러한 object를 수정했을 떄 cluster의 비저상적 행동을 유발할 수도 있다.

### Auto-reconciliation
kube-apiserver는 startup 시 기본 ClusterRole의 누락된 rule을 추가하고, ClusterRoleBinding에 누락된 주체를 추가한다. 이를 통해 cluster는 실수로 인한 수정 사항을 복구하고 새로운 k8s 릴리즈로 rule, 주체가 변경될 때 최신 상태로 유지할 수 있다.

이 자동 조정을 사용하지 않기 위해 기본 ClusterRole의, ClusterRoleBinding에 `rbac.authorization.kubernetes.io/autoupdate` annotation을 false로 설정한다. 물론 기본 rule, 주체가 누락되면 cluster가 비정상적으로 동작할 수 있다는 점에 유의해야 한다.

RBAC authorizer가 활성화되어 있는 경우 auto-reconciliation은 기본적으로 활성화된다.

### API discovery roles
기본 ClusterRoleBinding은 인증되지 않은 사용자와 인증된 사용자가 public 접근할 수 있는 것으로 간주되는 API 정보를 읽을 수 있도록 허용한다(crd 포함). anonymous unauthenticated 접근을 비활성화하기 위해 kube-apiserver에 `--anonymous-auth=false`를 추가한다.

kubectl 명령어를 사용해 확인할 수 있다.
``` sh
kubectl get clusterroles system:discovery -o yaml
```

아래는 출력 예시다.
``` yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  creationTimestamp: "2024-03-24T00:49:45Z"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:discovery
  resourceVersion: "70"
  uid: 528663ed-9432-4b9f-883e-78aa66500c4d
rules:
- nonResourceURLs:
  - /api
  - /api/*
  - /apis
  - /apis/*
  - /healthz
  - /livez
  - /openapi
  - /openapi/*
  - /readyz
  - /version
  - /version/
  verbs:
  - get
```

> **Note**:  
> If you edit that ClusterRole, your changes will be overwritten on API server restart via auto-reconciliation. To avoid that overwriting, either do not manually edit the role, or disable auto-reconciliation.

아래 표는 기본 제공되는 ClusterRole과 ClusterRole에 바인딩 된 subject에 대한 정보를 나타낸다. ClusterRoleBinding의 이름은 표에 없지만 직접 확인해보면 ClusterRole과 동일한 이름을 가진 것을 확인할 수 있다.
| Default ClusterRole<br>(ClusterRole) | Default ClusterRoleBinding<br>(ClusterRole에 바인딩된 subject 목록) | Description                                                                                                                                                                      |
|--------------------------------------|----------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| system:basic-user                    | system:authenticated group                                                 | Allows a user read-only access to basic information about themselves. Prior to v1.14, this role was also bound to system:unauthenticated by default.                             |
| system:discovery                     | system:authenticated group                                                 | Allows read-only access to API discovery endpoints needed to discover and negotiate an API level. Prior to v1.14, this role was also bound to system:unauthenticated by default. |
| system:public-info-viewer            | system:authenticated and system:unauthenticated groups                     | Allows read-only access to non-sensitive information about the cluster. Introduced in Kubernetes v1.14.                                                                          |

### User-facing roles
기본 ClusterRole 중 일부는 사용를 위한 role을 나타내기 위해 `system` 접두사를 갖지 않는 경우도 있다. `cluster-admin`은 ClusterRoleBinding과 사용해 cluster 전역에 대한 role, `admin`, `edit`, `view`는 RoleBinding과 사용해 특정 ns에 대한 role을 위한 것이다.

user-facing rule은 관리자가 cr에 대한 rule을 포함할 수 있도록 하기 위해 ClusterRole aggregation을 허용한다. `admin`, `edit`, `view` ClusterRole에 rule을 추가하기 위해 아래와 같은 label을 갖는 ClusterRole을 생성하면된다.
``` yaml
metadata:
  labels:
    rbac.authorization.k8s.io/aggregate-to-admin: "true"
    rbac.authorization.k8s.io/aggregate-to-edit: "true"
    rbac.authorization.k8s.io/aggregate-to-view: "true"
```

아래 표는 기본 제공되는 ClusterRole과 ClusterRole에 바인딩 된 subject에 대한 정보를 나타낸다. ClusterRoleBinding의 이름은 표에 없지만 직접 확인해보면 ClusterRole과 동일한 이름을 가진 것을 확인할 수 있다.
| Default ClusterRole<br>(ClusterRole) | Default ClusterRoleBinding<br>(ClusterRole에 바인딩된 subject 목록) | Description                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
|--------------------------------------|---------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| cluster-admin                        | system:masters group                                                | Allows super-user access to perform any action on any resource. When used in a ClusterRoleBinding, it gives full control over every resource in the cluster and in all namespaces. When used in a RoleBinding, it gives full control over every resource in the role binding's namespace, including the namespace itself.                                                                                                                                                                                                                                            |
| admin                                | None                                                                | Allows admin access, intended to be granted within a namespace using a RoleBinding.<br>If used in a RoleBinding, allows read/write access to most resources in a namespace, including the ability to create roles and role bindings within the namespace. This role does not allow write access to resource quota or to the namespace itself. This role also does not allow write access to EndpointSlices (or Endpoints) in clusters created using Kubernetes v1.22+. More information is available in the "Write Access for EndpointSlices and Endpoints" section. |
| edit                                 | None                                                                | Allows read/write access to most objects in a namespace.<br>This role does not allow viewing or modifying roles or role bindings. However, this role allows accessing Secrets and running Pods as any ServiceAccount in the namespace, so it can be used to gain the API access levels of any ServiceAccount in the namespace. This role also does not allow write access to EndpointSlices (or Endpoints) in clusters created using Kubernetes v1.22+. More information is available in the "Write Access for EndpointSlices and Endpoints" section.                |
| view                                 | None                                                                | Allows read-only access to see most objects in a namespace. It does not allow viewing roles or role bindings.<br>This role does not allow viewing Secrets, since reading the contents of Secrets enables access to ServiceAccount credentials in the namespace, which would allow API access as any ServiceAccount in the namespace (a form of privilege escalation).                                                                                                                                                                                                |

### Core component roles
아래 표는 기본 제공되는 ClusterRole과 ClusterRole에 바인딩 된 subject에 대한 정보를 나타낸다. ClusterRoleBinding의 이름은 표에 없지만 직접 확인해보면 ClusterRole과 동일한 이름을 가진 것을 확인할 수 있다.
| Default ClusterRole<br>(ClusterRole) | Default ClusterRoleBinding<br>(ClusterRole에 바인딩된 subject 목록) | Description                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
|--------------------------------------|---------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| system:kube-scheduler                | system:kube-scheduler user                                          | Allows access to the resources required by the scheduler component.                                                                                                                                                                                                                                                                                                                                                                                          |
| system:volume-scheduler              | system:kube-scheduler user                                          | Allows access to the volume resources required by the kube-scheduler component.                                                                                                                                                                                                                                                                                                                                                                              |
| system:kube-controller-manager       | system:kube-controller-manager user                                 | Allows access to the resources required by the controller manager component. The permissions required by individual controllers are detailed in the controller roles.                                                                                                                                                                                                                                                                                        |
| system:node                          | None                                                                | Allows access to resources required by the kubelet, including read access to all secrets, and write access to all pod status objects.<br>You should use the Node authorizer and NodeRestriction admission plugin instead of the system:node role, and allow granting API access to kubelets based on the Pods scheduled to run on them.<br>The system:node role only exists for compatibility with Kubernetes clusters upgraded from versions prior to v1.8. |
| system:node-proxier                  | system:kube-proxy user                                              | Allows access to the resources required by the kube-proxy component.                                                                                                                                                                                                                                                                                                                                                                                         |
### Other component roles
아래 표는 기본 제공되는 ClusterRole과 ClusterRole에 바인딩 된 subject에 대한 정보를 나타낸다. ClusterRoleBinding의 이름은 표에 없지만 직접 확인해보면 ClusterRole과 동일한 이름을 가진 것을 확인할 수 있다.
| Default ClusterRole<br>(ClusterRole) | Default ClusterRoleBinding<br>(ClusterRole에 바인딩된 subject 목록) | Description                                                                                                                                                                                                                                                                                                                               |
|--------------------------------------|---------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| system:auth-delegator                | None                                                                | Allows delegated authentication and authorization checks. This is commonly used by add-on API servers for unified authentication and authorization.                                                                                                                                                                                       |
| system:heapster                      | None                                                                | Role for the Heapster component (deprecated).                                                                                                                                                                                                                                                                                             |
| system:kube-aggregator               | None                                                                | Role for the kube-aggregator component.                                                                                                                                                                                                                                                                                                   |
| system:kube-dns                      | kube-dns service account in the kube-system namespace               | Role for the kube-dns component.                                                                                                                                                                                                                                                                                                          |
| system:kubelet-api-admin             | None                                                                | Allows full access to the kubelet API.                                                                                                                                                                                                                                                                                                    |
| system:node-bootstrapper             | None                                                                | Allows access to the resources required to perform kubelet TLS bootstrapping.                                                                                                                                                                                                                                                             |
| system:node-problem-detector         | None                                                                | Role for the node-problem-detector component.                                                                                                                                                                                                                                                                                             |
| system:persistent-volume-provisioner | None                                                                | Allows access to the resources required by most dynamic volume provisioners.                                                                                                                                                                                                                                                              |
| system:monitoring                    | system:monitoring group                                             | Allows read access to control-plane monitoring endpoints (i.e. kube-apiserver liveness and readiness endpoints (/healthz, /livez, /readyz), the individual health-check endpoints (/healthz/*, /livez/*, /readyz/*), and /metrics). Note that individual health check endpoints and the metric endpoint may expose sensitive information. |

### Roles for built-in controllers
kube-controller-manager에 `--use-service-account-credentials` flag를 사용하면 각 controller는 별도의 sa를 사용한다. 각 내장 contaoller에 대응하는 role은 `system:controller` 접두사를 갖는다. 만약 위 flag 없이 kube-controller-manager를 시작하면 모든 control loop는 자신의 credential을 사용하기 때문에 관련된 모든 role을 부여해야 한다. 아래는 role 목록이다.

아래는 ClusterRole 목록이며 ClusterRoleBinding의 이름은 표에 없지만 직접 확인해보면 ClusterRole과 동일한 이름을 가진 것을 확인할 수 있다. sa의 이름은 ClusterRoleBinding을 조회해 확인할 수 있다.
- `system:controller:attachdetach-controller`
- `system:controller:certificate-controller`
- `system:controller:clusterrole-aggregation-controller`
- `system:controller:cronjob-controller`
- `system:controller:daemon-set-controller`
- `system:controller:deployment-controller`
- `system:controller:disruption-controller`
- `system:controller:endpoint-controller`
- `system:controller:expand-controller`
- `system:controller:generic-garbage-collector`
- `system:controller:horizontal-pod-autoscaler`
- `system:controller:job-controller`
- `system:controller:namespace-controller`
- `system:controller:node-controller`
- `system:controller:persistent-volume-binder`
- `system:controller:pod-garbage-collector`
- `system:controller:pv-protection-controller`
- `system:controller:pvc-protection-controller`
- `system:controller:replicaset-controller`
- `system:controller:replication-controller`
- `system:controller:resourcequota-controller`
- `system:controller:root-ca-cert-publisher`
- `system:controller:route-controller`
- `system:controller:service-account-controller`
- `system:controller:service-controller`
- `system:controller:statefulset-controller`
- `system:controller:ttl-controller`

## Privilege escalation prevention and bootstrapping
RBAC API는 사용자가 Role, ClusterRole, RoleBinding, ClusterRoleBinding을 편집해 권한을 상승시키는 것을 허용하지 않는다. 이는 API 레벨에서 강제되는 것이기 때문에 RBAC authorizor를 사용하지 않더라도 적용된다.

### Restrictions on role creation or update
아래 한가지 내용을 만족하면 role을 생성, 업데이트할 수 있다.
1. role에 포함된 권한을 사용자가 가지고 있는 경우
2. `rbac.authorization.k8s.io` API 그룹 내 `roles`, `clusterroles` 리소스에 대한 `escalate` verb를 수행할 수 있는 권한이 명시적으로 있는 경우

예를 들어 `user-1`이 cluster 전역에 대한 secret을 나열할 권한이 없는 경우 해당 권한을 갖는 ClusterRole을 생성할 수 없다. 이를 가능하게 하기 위해
1. Role, ClusterRole을 생성. 업데이트할 수 있는 권한을 준다.
2. 생성 할 Role, ClusterRole에 포함될 권한을 사용자에게 준다.
    - 포함될 권한을 사용자에게 준다.
    - 또는 명시적으로, `rbac.authorization.k8s.io` API 그룹 내 `roles`, `clusterroles` 리소스에 대한 `escalate` verb를 수행할 수 있는 권한을 준다

### Restrictions on role binding creation or update

## Command-line utilities