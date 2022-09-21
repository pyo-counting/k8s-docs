**Note**:
psp는 k8s v1.21에서 deprecated 됐으며 v1.25에서 삭제됐다. psp를 사용하는 대신 아래 방법을 사용해 동일한 제한을 po에 적용할 수 있다.

- psa
- 3rd party admission plugin

For a migration guide, see Migrate from PodSecurityPolicy to the Built-In PodSecurity Admission Controller. For more information on the removal of this API, see PodSecurityPolicy Deprecation: Past, Present, and Future.

If you are not running Kubernetes v1.25, check the documentation for your version of Kubernetes.

## What is a Pod Security Policy?
psp는 po의 보안과 관련된 측면을 제어하는 클러스터 수준 resource다(non-namespaced). psp object는 관련 필드에 대한 기본 값 뿐만 아니라 k8s에서 승인되기 위해 po가 만족해야 하는 조건들을 정의한다. 관지라는 아래 목록을 제어할 수 있다:

|Control Aspec|Field|Names|
|-------------|-----------|

## Enabling Pod Security Policies
psp 제어는 옵션인 admission controller로 구현된다. psp는 admission controller를 활성화 함으로써 동작하며, 어떠한 정책도 인가하지 않고 활성화한다면 클러스터에서 어떠한 po도 생성할 수 없다.

psps API(policy/v1beta1/podsecuritypolicy)는 admission controller와 별개로 활성화되므로 기존 클러스터의 경우 admission controller를 활성화하기 전에 psp을 추가하고 인가해야 한다.

## Authorizing Policies
psp resource만 생성되면 어떤 것도 하지 않는다. 이를 사용하기 위해 요청 사용자 또는 sa에 정책을 사용할 수 있도록 인가해야 한다. 이는 psp에 use 동사를 사용함으로 써 인가할 수 있다.

대부분의 k8s po는 사용자에 의해 직접 생성되지 않으며, 대신 deploy, rs와 관련된 controller manager에서 간접적으로 생성한다. 이러한 controller에 권한을 인가함으로써 해당 controller가 생성한 모든 po에 권한을 인가한다. 따라서 선호되는 방법은 po의 sa에 정책을 인가하는 것이다.

사용자가 직접 po를 생성할 경우 사용자 계정에 인가된 정책이 적용되며 deploy, rs, rc, sts와 같이 controller에서 po를 생성할 경우 sa에 인가된 정책이 적용된다.

### Via RBAC
k8s의 기본 인가 모드인 RBAC를 사용해 정책의 사용을 인가할 수 있다.

먼저 ClusterRole(또는 Role)에 원하는 정책에 대한 사용(use)를 인가한다.

``` yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: <role name>
rules:
- apiGroups: ['policy']
  resources: ['podsecuritypolicies']
  verbs:     ['use']
  resourceNames:
  - <list of policies to authorize>
```

ClusterRole(또는 Role)를 특정 사용자에 바운드한다.

``` yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: <binding name>
roleRef:
  kind: ClusterRole
  name: <role name>
  apiGroup: rbac.authorization.k8s.io
subjects:
# Authorize all service accounts in a namespace (recommended):
- kind: Group
  apiGroup: rbac.authorization.k8s.io
  name: system:serviceaccounts:<authorized namespace>
# Authorize specific service accounts (not recommended):
- kind: ServiceAccount
  name: <authorized service account name>
  namespace: <authorized pod namespace>
# Authorize specific users (not recommended):
- kind: User
  apiGroup: rbac.authorization.k8s.io
  name: <authorized user name>
```

If a RoleBinding (not a ClusterRoleBinding) is used, it will only grant usage for pods being run in the same namespace as the binding. This can be paired with system groups to grant access to all pods run in the namespace:

``` yaml
# Authorize all service accounts in a namespace:
- kind: Group
  apiGroup: rbac.authorization.k8s.io
  name: system:serviceaccounts
# Or equivalently, all authenticated users in a namespace:
- kind: Group
  apiGroup: rbac.authorization.k8s.io
  name: system:authenticated
  ```

여러 psp를 사용하 수 있는 경우 psp controller는 다음 기준에 따라 정책을 선택한다:

1. PodSecurityPolicies which allow the pod as-is, without changing defaults or mutating the pod, are preferred. The order of these non-mutating PodSecurityPolicies doesn't matter.
2. If the pod must be defaulted or mutated, the first PodSecurityPolicy (ordered by name) to allow the pod is selected.

**Note**: During update operations (during which mutations to pod specs are disallowed) only non-mutating PodSecurityPolicies are used to validate the pod.

### Recommended Practice

### Troubleshooting

## Policy Order