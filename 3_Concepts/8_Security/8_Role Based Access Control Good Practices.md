k8s RBAC는 cluster 사용자, workload가 필요한 resource에만 접근할 수 있도록 하는 중요한 security control이다. cluster 관리자는 권한을 설계할 때 권한 상승(privilege escalation)이 발생할 수 있는 상황을 이해하고 과도한 접근으로 인한 보안 사고를 줄이는 것이 중요하다.

아래 good practice는 [RBAC ducumantion](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#restrictions-on-role-creation-or-update)과 같이 읽는 것을 권장한다.

## General good practice
### Least privilege
이상적으로는 사용자와 sa에게 최소한의 RBAC 권한(필요한 권한)만 부여해야 한다. 일반적인 규칙은 다음과 같다.
- 가능한 한 ns 레벨에서 권한을 할당한다. 사용자에게 특정 ns 내에서만 권한을 부여하기 위해 RoleBinding을 사용하고 ClusterRoleBinding을 되도록 사용하지 않는다.
- 가능하면 와일드카드(*) 권한을 제공하지 않는다(특히 모든 resource를 나타내는 경우). k8s는 확장 가능한 시스템이므로 와일드카드 접근 권한을 부여하면 현재 cluster에 존재하는 모든 object 뿐만 아니라 추후 추가될 모든 object도 포함하는 것을 의미한다.
- 관리자는 필요할 경우에만 `cluster-admin` ClusterRole을 사용해야 한다. 낮은 수준의 권한을 갖으면서 [impersonation rights](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#user-impersonation)을 사용하면 cluster resource에 대한 수정으로 이한 사고를 방지할 수 있다.
- 사용자를 `system:masters` group에 추가하지 않는다. 이 그룹의 구성원은 모든 RBAC 권한 검사를 우회하고 언제나 슈퍼유저 접근 권한을 가지며 RoleBindings나 ClusterRoleBindings를 제거해도 권한을 철회할 수 없다. 추가로 cluster가 authorization webhook을 사용하는 경우 이 그룹의 구성원은 웹훅도 우회한다(이 그룹의 구성원으로부터의 요청은 웹훅으로 전송되지 않는다).

### Minimize distribution of privileged tokens
이상적으로는 높은 레벨의 권한이 부여된 sa을 po에 할당하면 안된다(예를 들어, [privilege escalation risks](https://kubernetes.io/docs/concepts/security/rbac-good-practices/#privilege-escalation-risks)에 나열된 권한). workload가 높은 레벨의 권한을 필요로 하는 경우 아래 사항을 고려한다.
- 높은 수준의 권한을 갖는 po를 실행하는 no의 수를 제한한다. 실행 중인 ds가 필요한지 확인하고, 최소한의 권한을 갖고있음으로써 blast radius of container escape를 제한한다.
- 신뢰할 수 없는 또는 공개적으로 노출된 po와 높은 수준의 권한을 갖는 po를 같이 실행하지 않는다. [Taints and Toleration](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/), [NodeAffinity](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#node-affinity), [PodAntiAffinity](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#inter-pod-affinity-and-anti-affinity)를 사용해 신뢰할 수 없는 po와 함께 실행되지 않도록 한다. 신뢰할 수 없는 po가 Restricted Pod Security Standard를 충족하지 않는 경우에 특히 주의한다.

### Hardening
k8s는 기본적으로 cluster에서 필요하지 않을 수 있도 있는 접근 권한을 기본적으로 제공한다. 기본 제공되는 RBAC 권한을 검토해 보안을 강화할 수 있다. 일반적으로, `system:` ClusterRole에 제공되는 권한을 변경해서는 안 되지만 cluster 권한 강화를 위한 몇가지 옵션이 있다.
- `system:unauthenticated` 그룹에 대한 binding을 검토하고 가능한 경우 제거한다. 이는 네트워크 레벨에서 kube-apiserver에 접속할 수 있는 모든 사람에게 접근을 제공하기 때문이다.
- sa의 `.automountServiceAccountToken`를 false로 설정해 sa의 token이 기본적으로 mount되는 것을 피한다. 자세한 내용은 [using default service account token](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#use-the-default-service-account-to-access-the-api-server)을 참고한다.

### Periodic review
주기적으로 k8s RBAC 설정을 검토하고 중복 항목과 잠재적인 권한 상승을 점검하는 것이 중요하다. 공격자가 삭제된 사용자와 동일한 이름의 사용자 계정을 만들 수 있다면 삭제된 사용자의 모든 권한을 자동으로 상속할 수 있다.

## Kubernetes RBAC - privilege escalation risks
k8s RBAC 내에는 사용자나 sa가 cluster 내에서 권한을 상승시키거나 cluster 외부의 시스템에 영향을 줄 수 있는 권한들이 있다.

아래에서는 cluster 관리자가 의도치 않게 더 많은 접근 권한을 부여하지 않도록 하기위한 주의 사항을 살펴본다.

### Listing secrets
사용자에게 secret에 대한 `get` 접근 권한을 부여하면 내용을 읽을 수 있게 된다. 또한 `list`, `watch` 접근 권한도 사용자에게 secret의 내용을 공개할 수 있다. 예를 들어 `kubectl get secrets -A -o yaml` 명령어를 사용하면 모든 secret의 내용을 응답한다.

### Workload creation
ns 내에서 workload(po, po를 관리하는 [workload resource](https://kubernetes.io/docs/concepts/workloads/controllers/))를 생성할 수 있는 권한을 부여하면 해당 ns 내의 많은 다른 resource(예를 들어 po에 mount될 수 있는 secret, cm, pv)에 대한 접근 권한이 암묵적으로 부여되는 것이다. 그리고 po는 모든 sa를 사용할 수 있기 때문에 때문에 workload를 생성할 수 있는 권한을 부여하면 해당 ns의 모든 sa가 갖는 권한도 부여되는 것이다.

privileged po를 실행할 수 있는 사용자는 해당 접근 권한을 사용해 no 접근 권한을 얻고 잠재적으로 자신의 권한을 더 높일 수 있다. Where you do not fully trust a user or other principal with the ability to create suitably secure and isolated Pods, you should enforce either the Baseline or Restricted Pod Security Standard. You can use [Pod Security admission](https://kubernetes.io/docs/concepts/security/pod-security-admission/) or other (third party) mechanisms to implement that enforcement.

이러한 이유로 ns는 서로 다른 신뢰 수준이나 tenancy가 필요한 resource를 분리하는 데 사용해야 한다. [least privilege](https://kubernetes.io/docs/concepts/security/rbac-good-practices/#least-privilege) 원칙을 따르고 최소한의 권한을 할당하는 것이 최선의 방법이긴하지만 ns 안에서의 경계는 약한 것으로 간주된다.

### Persistent volume creation
누군가(또는 애플리케이션)가 pv를 생성할 수 있도록 허용하면 `hostPath` volume을 생성할 수 있는 권한도 허용하는 것이다. 그러면 해당 po를 통해 관련 no의 호스트 파일시스템에 접근할 수 있게 된다. 

호스트 파일 시스템에 접근할 수 있는 container는 다른 container의 데이터를 읽고 kubelet과 같은 시스템 서비스의 credential을 악용하는 등 여러 가지 방법으로 권한을 상승시킬 수 있다.

pv object를 생성할 수 있는 권한은 다음의 경우에만 부여해야 한다.
- 이 접근이 필요한 작업을 수행하는 사용자(cluster 운영자)와 신뢰할 수 있는 사용자.
- automatic provisioning을 위해 pvc를 기반으로 pv를 생성하는 k8s control plane 구성 요소. 이는 보통 k8s provider나 CSI driver 설치 시 관리자가 설정한다.

persistent storage가 필요한 경우 신뢰할 수 있는 관리자가 pv를 생성하고, 제한된 사용자는 pvc를 사용헤 스토리지에 접근해야 한다.

### Access to `proxy` subresource of Nodes
no object의 sub-resource인 proxy에 대한 접근 권한이 있는 사용자는 kubelet API에 대한 권한을 갖는 것이다. 즉, kubelet API를 사용해 해당 no의 모든 po에서 명령어를 실행할 수 있다. 이는 audit logging과 admission control을 우회하기 때문에 권한 부여에 신중해야 한다.

### Escalate verb
일반적으로 RBAC 시스템은 사용자가 소유한 권한보다 더 많은 권한을 가진 ClusterRole를 생성하는 것을 방지한다. 이에 대한 예외로 `escalate` verb가 있다. [RBAC documentation](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#restrictions-on-role-creation-or-update)과 같이 이 권한을 가진 사용자는 실질적으로 자신의 권한을 상승시킬 수 있다.

### Bind verb
`escalate` verb와 유사하게, 이 권한을 사용자에게 부여하면 사용자가 이미 가지고 있지 않은 권한을 가진 role에 binding을 생성할 수 있다.

### Impersonate verb
이 verb는 사용자가 cluster 내 다른 사용자를 가장하고 해당 사용자의 권한을 가질 수 있게 한다. 이 권한을 부여할 때는 신중해야 하며, 가장된 계정을 통해 과도한 권한이 부여되지 않도록 해야 한다.

### CSRs and certificate issuing
CSR API를 통해 csr을 생성할 수 있는 권한과 `certificatesigningrequests/approval`에 대한 업데이트 권한이 있는 사용자는 `kubernetes.io/kube-apiserver-client` signer가 있는 경우 cluster에 인증할 수 있는 새로운 클라이언트 인증서를 생성할 수 있다. 이러한 클라이언트 인증서는 k8s 시스템 구성 요소와 중복되는 임의의 이름을 가질 수 있기 때문에 실질적으로 권한 상승을 허용하게 되는 것이다.

### Token request
`serviceaccounts/token`에 대한 생성 권한이 있는 사용자는 기존 sa에 대한 tokne을 발급하는 API를 사용할 수 있다.

### Control admission webhooks
Users with control over `validatingwebhookconfigurations` or `mutatingwebhookconfigurations` can control webhooks that can read any object admitted to the cluster, and in the case of mutating webhooks, also mutate admitted objects.

### Namespace modification
ns object에 patch 작업을 수행할 수 있는 사용자는 ns의 label을 수정할 수 있다. Pod Security Admission이 사용되는 cluster에서는 이를 통해 사용자가 관리자가 의도한 것보다 더 허용적인 정책으로 ns를 구성할 수 있다. NetworkPolicy가 사용되는 cluster에서는 사용자가 label을 설정해 관리자가 허용하지 않은 서비스에 대한 접근을 간접적으로 허용할 수 있다.

## Kubernetes RBAC - denial of service risks
### Object creation denial-of-service
cluster에서 object를 생성할 수 있는 권한이 있는 사용자는 충분히 큰 object를 생성해 denial of service 상태를 만들 수 있다. 이는 [etcd used by kubernetes is vulnerable to OOM attac](https://github.com/kubernetes/kubernetes/issues/107325)와 관련이 있다. 이는 반신뢰 또는 비신뢰 사용자에게 제한된 접근 권한이 허용되는 multi-tenant cluster과 특히 관련이 있을 수 있다.

이 문제를 완화하는 한 가지 방법은 생성할 수 있는 object의 수량을 제한하는 [resource quotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/#object-count-quota)을 사용하는 것이다.