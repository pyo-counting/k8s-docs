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
RoleBinding은 Role에 정의된 권한을 사용자나 사용자 그룹, sa 등의 주체(subject)에게 부여한다. RoleBinding은 주체(subjects) 목록과 부여된 Role에 대한 참조 정보를 갖는다. RoleBinding은 특정 ns 내에서 권한을 부여하는 반면, ClusterRoleBinding은 클러스터 전역에서 해당 권한을 부여한다.

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

#### Role examples

### Referring to subjects