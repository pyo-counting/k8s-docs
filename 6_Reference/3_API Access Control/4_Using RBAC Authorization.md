RBAC는 조직 내 개별 사용자의 역할에 따라 컴퓨터 또는 네트워크 리소스에 대한 접근을 규제하는 방법이다.

RBAC 인가는 rbac.authorization.k8s.io API 그룹을 사용하여 인가 결정을 내리므로 k8s API를 통해 정책을 동적으로 설정할 수 있다.

RBAC를 활성화하려면 --authorization-mode flag 목록에 RBAC를 포함해 API server를 시작한다. 예를 들어

``` bash
kube-apiserver --authorization-mode=Example,RBAC --other-options --more-options
```

## API objects
RBAC API는 Role, ClusterRole, RoleBinding ClusterRoleBinding 네 가지 k8s resource를 통해 구현된다. 다른 k8s resource와 마찬가지로 kubectl과 같은 도구를 사용해 다룰 수 있다.

**Caution**: These objects, by design, impose access restrictions. If you are making changes to a cluster as you learn, see privilege escalation prevention and bootstrapping to understand how those restrictions can prevent you making some changes.

### Role and ClusterRole
RBAC Role과 ClusterRole은 권한 집합을 나타내는 규칙이 포함한다. 권한을 허용하는 방식으로 구현된다(권한에 대해 "부정"하는 방식으로 구현되지 않음).

Role은 항상 특정 ns 내에서 권한을 설정한다; Role을 생성할 때 Role이 속한 ns를 설정해야 한다.

이와 반대로 ClusterRole은 non-namespaecd resource다. k8s object는 namespaced 또는 non-namespaced이기 때문에 이를 구분하기 위해 2개의 rule이 있는 것이다(둘 다 될수는 없다).

ClusterRole과 관련에 몇 가지 사용사례가 있다:

1. namespaced resource에 대한 권한을 정의하고 각 ns 내에서 접근 권한을 부여한다.
2. namespaced resource에 대한 권한을 정의하고 모든 ns에 대한 접근 권한을 부여 받는다.
3. cluster-scoped resource에 대한 권한을 정의한다.

ns 내에서 역할을 정의할 경우 Role을 사용하고 클러스터에서 역할을 정의할 경우 ClusterRole을 사용한다.

#### Role example

#### ClusterRole example
ClusterRole을 사용해 Role과 동일한 권한을 부여할 수 있다. ClusterRole은 cluster-scoped로 아래 목록에 대한 접근 권한을 부여할 수도 있다:

- cluster-scoped resources (예를 들어 no)
- non-resource endpoints (예를 들어 /healthz)
- 모든 ns에 대한 namespaced resources(예를 들어 po). 예를 들어 kubectl get po -A 명령어를 사용하기 위해 ClusterRole을 사용할 수 있다.

**Note**: 위 설명만 보면 non-namespaced resource, non-resource들에 대한 권한은 ClusterRole에만 권한을 정의할 수 있을 것 같지만 Role에도 부여할 수 있는 것 같다. 그렇다면 ClusterRole은 여러 ns에서 공유하기 위한 역할만 하는 것뿐인가?