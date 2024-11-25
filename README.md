## about docs
- v1.31 버전 기준 작성

## configure cluster
### general
- k8s의 모든 구성 요소를 k8s cluster 내에서 container로 관리하는 것을 권장한다. 하지만 container 실행을 담당하는 kubelet은 container로 실행할 수 없다. ([Getting started](https://kubernetes.io/docs/setup/))
- k8s의 control plane은 linux에서 실행되도록 디자인됐다. 물론 cluster 내에서 애플리케이션은 windows를 포함한 다른 os에서 실행할 수도 있다. ([Getting started](https://kubernetes.io/docs/setup/#what-s-next))
- ns의 limit, hierarchical limit을 고려한다. ([Production environment](https://kubernetes.io/docs/setup/production-environment/#set-limits-on-workload-resources))
- k8s는 내부적으로 유저 인증 정보를 저장하지 않는다. 대부분의 웹 서비스나 인증 서버들은 사용자 정보를 내부적으로 저장하여 사용자로부터 인증 정보를 전달 받았을 때 저장된 정보를 바탕으로 인증을 처리한(예를 들어 웹 사이트에서 계정과 비밀번호를 입력 받아 유저DB를 조회하여 사용자 인증을 처리). k8s는 이와 다르게 따로 인증 정보를 저장하지 않고 외부 인증 시스템에서 제공해주는 신원 확인 기능들을 활용하여 사용자 인증을 하고 유저를 인식(identify)한다. 이러한 특징으로 인해 k8s에서는 쉽게 인증체계를 확장할 수 있다. k8s 내부 인증체계에 종속되는 부분이 거의 없기 때문이다. k8s는 사용자 인증체계를 전부 외부 시스템 (혹은 메커니즘)에 의존한다고 볼 수 있다. (X.509, HTTP Auth, Proxy Authentication 등) ([PKI certificates and requirements](https://kubernetes.io/docs/setup/best-practices/certificates/))
- k8s는 구성 요소간의 안전한 통신, 인증을 위해 PKI x.509 server, certificate 인증서를 사용한다. 기본적으로 kube-apiserver, etcd, front-proxy(옵션)를 위한 ica가 필요하다. ([PKI certificates and requirements](https://kubernetes.io/docs/setup/best-practices/certificates/#configure-certificates-manually))
- k8s v1.25버전부터 kube-apiserver가 server side field validation을 제공한다. 이는 kubectl의 --validation flag와 동일한 기능을 수행한다. ([Objects In Kubernetes](https://kubernetes.io/docs/concepts/overview/working-with-objects/#server-side-field-validation))
- 각 object는 resource 유형 별로 고유한 name을 갖는다. 또한 모든 k8s object는 cluster lifecycle 동안 고유한 UID를 갖는다. resource 내에서 동일한 name을 갖는 object가 존재할 수 없다. 물론 해당 object를 삭제하고 동일한 이름을 갖는 새로운 object를 생성할 수 있다(이 때 UUID는 다름). name은 동일한 resource에 대해 모든 API 버전(`.apiVersion` 필드)에서 유일해야 한다. API resource는 API group, resource type, namespace(namespaced resource일 경우)로 구분된다. 즉, API 버전은 상관이 없다. ([Object Names and IDs](https://kubernetes.io/docs/concepts/overview/working-with-objects/names/#names))
- 사용자가 설정하는 name은 resource URL에서 `/api/v1/pods/<name>`와 같이 object를 참조하는 데 사용된다. 가장 보수적으로 RFC 1035를 준수하는 것이 편하다. resource 마다 object naming 규칙에 대한 제약 사항이 더 많을 수도 있다. ([Object Names and IDs](https://kubernetes.io/docs/concepts/overview/working-with-objects/names/#rfc-1035-label-names))
- label은 k8s에서 기본 중요 grouping으로 사용된다. label key는 /로 구분된 prefix(optional), name 형태로 구성된다. `kubernetes.io/`, `k8s.io/` 접두사는 k8s core system에서 사용되도록 예약됐다. ([Labels and Selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/#syntax-and-character-set))
- label selector는 2개의 타입(equality-based, set-based)을 지원(`kubectl get -l`)한다. ([Labels and Selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/#label-selectors))
- ns는 여러 사용자가 있는 환경에서 여러 resource의 분리를 위해 사용한다. ns와 quota를 resource 사용에 대한 정책을 관리할 수 있다. 운영 환경에서는 default ns를 사용하는 것을 권장하지 않는다. ([Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/#when-to-use-multiple-namespaces))
- k8s cluster에는 `default`, `kube-system`, `kube-node-lease`, `kube-public` 기본 ns가 존재한다. ns에 대해 `kube-` 접두사는 k8s 시스템을 위해 예약됐기 때문에 사용하지 않는다. ([Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/%2523working-with-namespaces))
- svc 생성 시 ns의 이름을 사용해 cluster 내에서 domain을 생성한다(포맷은 `<service-name>.<namespace-name>.svc.cluster.local`). TLD와 동일한 이름을 갖는 ns의 경우 public TLD를 덮어쓸 수 있기 때문에 위험하다. 그렇기 때문에 제한된 사용자만 ns를 만들 수 있도록 제한한다. ([Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/#namespaces-and-dns))
- annotation key는 /로 구분된 prefix(optional), name 형태로 구성된다. `kubernetes.io/`, `k8s.io/` 접두사는 k8s core system에서 사용되도록 예약됐다. ([Annotations](https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/#syntax-and-character-set))
- 각 resource의 필드 필터링을 위해 field selector를 사용(`kubectl get --field-selector`)할 수 있다. 모든 resource가 공통적으로 `metadata.name`, `metadata.namespace` 필드를 지원하며, resource 종류에 따라 지원하는 추가 필드가 다르다. ([Field Selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/field-selectors/))
- finalizer는 삭제 마킹된 resource를 완전히 삭제하기 전에 특정 조건이 충족될 때까지 대기하도록 k8s에 지시하는 namespaced key다. finalizer는 일반적으로 실행항 코드를 지정하지 않는다. 대신 일반적으로 annotation과 유사한 특정 resource에 대한 key 목록이다. finalizer가 존재하는 resource를 삭제할 동작은 다음과 같다. ([Finalizers](https://kubernetes.io/docs/concepts/overview/working-with-objects/finalizers/))
  - 해당 resource를 삭제하려고 할 때 kube-apiserver는 finalizer 필드 값을 확인하고 다음을 수행한다.
    - 삭제 요청 시간을 `.metadata.deletionTimestamp` 필드 값을 추가해 ojbect를 수정한다.
    - `metadata.finalizers` 필드가 빈 상태가 될때까지 object가 삭제되지 않도록 한다.
    - HTTP 202 status code(Accepted)를 반환한다.
  - finalizer를 관리하는 controller는 object 삭제가 요청됐음을 나타내는 `.metadata.deletionTimestamp` 필드 설정에 대한 업데이트를 확인한다.
  - controller는 해당 resource에 대해 finalizer의 요구 사항을 충족하려고한다. finalizer 조건이 만족될 때마다 controller는 resource의 finalizer 필드에서 해당 키를 삭제한다. finalizer 필드가 빈 값이 되면 deletionTimestamp가 설정된 object가 자동으로 삭제(gc)된다.
- label(`metadata.labels`), finalizer(`.metadata.finalizers`), owner reference(`metadata.ownerReferences`)는 비슷해 보이지만 각기 다른 목적으로 사용된다. label과 owner reference는 모두 객체간의 관계를 나타내지만 label의 경우 controller가 관련있는 객체 들에 대한 변화를 추적할 때 사용하는 반면, owner reference는 관련 객체를 삭제할 때 종속 관계를 파악하고 cascading deletion(background, foreground)를 수행하는 데 사용한다. 이 삭제 단계에서 k8s는 finalizer 필드도 같이 고려한다. ([Finalizers](https://kubernetes.io/docs/concepts/overview/working-with-objects/finalizers/#owners-labels-finalizers))
- owner reference는 resource 간 소유권 관게를 정의(예를 들어 rs와 po)하고 부모 resource 삭제 시 자식 resource를 삭제하는 생명주기 메커니즘을 갖는다. 이에 반해 finalizer는 resource의 의존성이나 사용 중인 상태를 표현하는 데 사용된다(예를 들어 pv와 po의 관계). ([ChatGPT]())
- cross-namespace owner reference는 허용되지 않는다. namespaced object는 cluster-scoped, namespaced owner를 가질 수 있다. 반면 cluster-scoped object는 cluster-scoped owner만 가질 수 있다. ([Owners and Dependents](https://kubernetes.io/docs/concepts/overview/working-with-objects/owners-dependents/#owner-references-in-object-specifications))
- `app.kubernetes.io` prefix를 갖는 label을 관리하는 것을 권장한다. ([Recommended Labels](https://kubernetes.io/docs/concepts/overview/working-with-objects/common-labels/))
- k8s control plane의 핵심은 kube-apiserver다. kube-apiserver는 HTTP API를 노출하여 최종 사용자, cluster의 다양한 부분, 외부 구성 요소가 서로 통신할 수 있도록한다. k8s API를 사용하면 k8s에서 API object의 state를 조회하고 수정할 수 있다(예: po, ns, cm 등). 대부분의 작업은 kubectl, kubeadm과 같은 CLI를 통해 수행된다. 이러한 도구들은 k8s API를 사용한다. 물론 REST call을 사용해 API에 직접 접근할 수도 있다. k8s는 k8s API를 사용하여 애플리케이션을 개발하려는 사람들을 위한 client libraries를 제공한다. ([The Kubernetes API](https://kubernetes.io/docs/concepts/overview/kubernetes-api/#openapi-interface-definition))
- k8s는 'hub-and-spoke` API 패턴을 사용한다. control plane 구성 요소 중 kube-apiserver만 HTTPS 443 listen port를 노출하며 no, po는 kube-apiserver와만 통신한다. kube-apiserver는 1개 이상의 authentication, authorization을 설정해야 한다. po는 sa를 사용해 kube-apiserver와 통신하며 root certificate와 bearer token을 po에 주입한다. ([Communication between Nodes and the Control Plane](https://kubernetes.io/docs/concepts/architecture/control-plane-node-communication/#node-to-control-plane))
- default ns의 kubernetes svc는 kube-apiserver의 HTTPS 엔드포인트로 redirect(kube-proxy가 수행)되는 virtual ip로 구성되어 있다. ([Communication between Nodes and the Control Plane](https://kubernetes.io/docs/concepts/architecture/control-plane-node-communication/#node-to-control-plane))

- 각 ns에 존재하는 default sa는 사용자가 삭제하더라도 control plane이 재생성한다. ([Service Accounts](https://kubernetes.io/docs/concepts/security/service-accounts/#default-service-accounts))
-  k8s v1.22부터는 기본적으로 `TokenRequest` API를 사용해 짧은 수명의 automatically rotating token을 얻고 token을 projected volume으로 mount한다. token이 만료되면 kubelet은 token을 재발급 받는다. ([Service Accounts](https://kubernetes.io/docs/concepts/security/service-accounts/#assign-to-pod))
- sa에 `kubernetes.io/enforce-mountable-secrets` annotation을 추가해 sa에서 사용할 수 있는 secret 목록을 제어할 수 있다. ([Service Accounts](https://kubernetes.io/docs/concepts/security/service-accounts/#enforce-mountable-secrets))
- 사용자를 `system:masters` group에 추가하지 않는다. 이 그룹의 구성원은 모든 RBAC 권한 검사를 우회하고 언제나 슈퍼유저 접근 권한을 가지며 RoleBindings나 ClusterRoleBindings를 제거해도 권한을 철회할 수 없다. ([Role Based Access Control Good Practices](https://kubernetes.io/docs/concepts/security/rbac-good-practices/#general-good-practice))
- authentication을 정상적으로 통과하면 `system:authenticated` group에 속하게 된다. anonymous request authentication이 활성화된 경우 모든 authentication mode에 의해 거절되지 않는다면 사용자는 `system:anonymous` 이름을 갖고 `system:unauthenticated` 그룹에 속하게 된다.
- sa는 사용자 이름 `system:serviceaccount:(NAMESPACE):(SERVICEACCOUNT)`로 인증하고 `system:serviceaccounts`, `system:serviceaccounts:(NAMESPACE)` 그룹에 할당된다.
- bootstrap token authentication mode를 통해 인증된 사용자는 `system:bootstrap:<Token ID>` 이름을 갖고 `system:bootstrappers` 그룹에 속하게 된다.
- `system:unauthenticated` 그룹에 대한 binding을 검토하고 가능한 경우 제거한다. 이는 네트워크 레벨에서 kube-apiserver에 접속할 수 있는 모든 사람에게 접근을 제공하기 때문이다. ([Role Based Access Control Good Practices](https://kubernetes.io/docs/concepts/security/rbac-good-practices/#hardening))
- k8s non-resource에 대한 API와 resource에 대한 API(core group(`/api/v1`), name graoup(`/apis/${group}/${version}`))로 나뉜다. non-resource API는 HTTP 소문자 method를 사용해 authorization를 수행하며 resource API는 HTTP 소문자 method, 요청 resource 종류에 따라 request verb를 매핑하고 authorization을 수행한다. ([API Overview](https://kubernetes.io/docs/reference/using-api/), [Kubernetes API Concepts](https://kubernetes.io/docs/reference/using-api/api-concepts/), [Authorization](https://kubernetes.io/docs/reference/access-authn-authz/authorization/#determine-the-request-verb))
- kubelet, 사용자가 사용하는 kubeconfig 파일은 kubectl 명령어를 사용해 생성할 수 있다.

### control plane
- 구성 요소: `kube-apiserver`, `etcd`, `kube-scheduler`, `kube-controller-manager`, `cloud-controller-manager`
- control plane의 구성 요소 etcd는 3개 이상(raft 알고리즘을 위해)의 failure zone에서 실행하는 것을 권장한다. 이외 kube-controller-manager, cloud-controller-manager, kube-scheduler는 고가용성은 유지하지만 작업 충돌 방지를 위해 leader election 메커니즘을 사용하며 동일하게 3개 이상의 failure zone에서 실행하는 것은 권장한다. kube-apiserver는 load balancer 뒤에서 다중 운영이 가능하다. ([Production environment](https://kubernetes.io/docs/setup/production-environment/#production-control-plane), [Running in multiple zones](https://kubernetes.io/docs/setup/best-practices/multiple-zones/#control-plane-behavior))
- 대규모 cluster의 경우 Event object 저장을 위한 별도의 etcd를 운영을 고려한다. ([Considerations for large clusters](https://kubernetes.io/docs/setup/best-practices/cluster-large/#etcd-storage))
- kube-apiserver -> kubelet 통신 시, 기본적으로 kube-apiserver는 kubelet의 server certificate를 검증하지 않는다. 검증을 위해 `--kubelet-certificate-authority` flag에 kubelet의 ca certificate를 설정해 kubelet으로의 연결을 안전하게 수행할 수 있다(SSH tunneling을 대신 사용 가능). kube-apiserver -> kubelet 통신 경우는 다음과 같다. kube-apiserver가 kubelet에 no, po 상태에 대한 추가 정보 요청 등, po의 로그조회 (`kubectl logs`), 실행 중인 po에 대한 attach(`kubectl exec`), kubelet의 port-forwarding 일부 과정 중 po와 no의 정보를 식별할 때(`kubectl port-forward`) ([Communication between Nodes and the Control Plane](https://kubernetes.io/docs/concepts/architecture/control-plane-node-communication/#control-plane-to-node))
- kube-apiserver -> no, po, svc와 직접 통신하는 경우는 kube-apiserver의 proxy 기능을 사용할 때다. proxy 기능은 kube-apiserver의 내장 기능으로 kube-apiserver를 통해 po, svc, no에 접근하는 경우에 사용된다. 대표적인 예시로 `kubectl cluster-info` 명령어를 사용해 조회되는 목록이다. 해당 목록에는 k8s kube-apiserver의 도메인과 `kubernetes.io/cluster-servic` label이 true인 svc에 접근하기 위한 kube-apiserver의 주소를 출력한다. 물론 해당 목록에 조회되지 않더라도 kuber-apiserver의 API 명세서를 참고해 접근할 수 있다. ([Communication between Nodes and the Control Plane](https://kubernetes.io/docs/concepts/architecture/control-plane-node-communication/#api-server-to-nodes-pods-and-services))

- control plane 구성 요소(예를 들어 kube-scheduler, kube-controller-manager)의 여러 replica는 kube-system ns에 저장된 lease object를 사용해 reader를 관리한다 ([Leases](https://kubernetes.io/docs/concepts/architecture/leases/#leader-election))
- kube-apiserver는 kube-system ns에 저장된 lease object를 통해 k8s 전체 시스템에 kube-apiserver에 대한 정보를 제공한다 ([Leases](https://kubernetes.io/docs/concepts/architecture/leases/#api-server-identity))
- encryption at rest 고려 ([Security](https://kubernetes.io/docs/concepts/security/#control-plane-protection))
- kube-controller-manager
  - no의 non-graceful shutdown 처리를 위한 `NodeOutOfServiceVolumeDetach` feature gate 활성화 여부 확인. ([Node Shutdowns](https://kubernetes.io/docs/concepts/cluster-administration/node-shutdown/#non-graceful-node-shutdown))
  - no의 non-graceful shutdown 처리 대신 po의 삭제가 6분동안 실패할 경우 강제로 volume mount를 해제하는 `--disable-force-detach-on-timeout` 설정 확인. ([Node Shutdowns](https://kubernetes.io/docs/concepts/cluster-administration/node-shutdown/#storage-force-detach-on-timeout))
  - bootstrap token authentication mode 사용시 만료된 token을 gc하기 위해 `--controllers=tokencleaner,...` flag를 사용해 tokencleaner controller를 활성화한다. ([Authenticating](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#bootstrap-tokens))
- kube-apiserver

  - 적어도 사용자, sa 인증을 위한 2가지 authentication mode를 활성화 해야한다. authentication mode의 실행 순서는 랜덤이다. ([Authenticating](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#authentication-strategies))
  - k8s 내장 authentication은 사용자 인증에 적합하지 않기 때문에 oidc와 같은 외부 authentication mode를 사용해야 한다. ([Hardening Guide - Authentication Mechanisms](https://kubernetes.io/docs/concepts/security/hardening-guide/authentication-mechanisms/))
    - client certificate authentication mode 활성화는 `--client-ca-file` flag를 사용한다. ([Authenticating](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#x509-client-certificates))
    - static token authentication mode 활성화는 `--token-auth-file` flag를 사용한다. ([Authenticating](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#static-token-file))
    - bootstrap token authentication mode 활성화는 `--enable-bootstrap-token-auth` flag를 사용한다. 이때 bootstrap token gc를 위해 kube-controller-manager에 tokencleaner controller를 활성화 해야한다. ([Authenticating](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#bootstrap-tokens))
    - bootstrap token을 사용해 kubelet이 client certificate를 발급받아야한다. 이 때 k8s에 내장된 `kubernetes.io/kube-apiserver-client-kubelet` signer에 의해 자동 승인 및 서명되기 위해서는 kube-controller-manager에 `--cluster-signing-cert-file` flag, `--cluster-signing-key-file` flag를 사용해야한다. 이 내장 signer는 csr을 요청한 credential의 승인에 대한 권한 유무(SubjectAccessReview API를 사용해 확인)에 따라 승인을 수행함으로써 자동 승인을 구현한다. ([TLS bootstrapping](https://kubernetes.io/docs/reference/access-authn-authz/kubelet-tls-bootstrapping/#access-to-key-and-certificate), [Certificates and Certificate Signing Requests](https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests/#approval-rejection-control-plane))
    - sa token authentication mode 활성화는 `--service-account-key-file` flag를 사용한다. kube-controller-manager에는 `--service-account-private-key-file` flag를 통해 개인키 파일을 설정한다. ([Authenticating](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#service-account-tokens), [PKI certificates and requirements](https://kubernetes.io/docs/setup/best-practices/certificates/#certificate-paths))
    - oidc token, webhook token, authentication proxy authentication mode 활성화는 [Authenticating](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#openid-connect-tokens) 참고
    - kubectl은 webhook, oidc provider로부터 사용자 credential 획득을 위해 client-go credential plugin을 사용할 수 있다.
    - eks의 경우 내부적으로 aws-iam-authenticator oss를 webhook token authenticator로 사용한다.
    - anonymous request authentication mode 활성화는 `--anonymous-auth` flag를 사용한다. ([Authenticating](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#anonymous-requests))
  - authorization mode 활성화는 `--authorization-mode` flag를 사용한다. 나열된 순서대로 authorization을 진행한다. ([Authorization](https://kubernetes.io/docs/reference/access-authn-authz/authorization/#authorization-modules))
    - node authorizor에 의해 인증되기 위해 kubelet은 system `system:node:<nodeName>` 사용자, `system:nodes` 그룹에 대한 credentials을 사용해야 한다. 해당 사용자, 그룹 형식은 [kubelet TLS bootstrapping](https://kubernetes.io/docs/reference/access-authn-authz/kubelet-tls-bootstrapping/)을 통해 생성된 kubelet의 identity와 일치한다. ([Using Node Authorization](https://kubernetes.io/docs/reference/access-authn-authz/node/))

### data plane (node)
- 구성 요소: `kubelet`, `container runtime`, `kube-proxy`
- no 모니터링을 위한 node problem detector 설치 고려 ([Production environment](https://kubernetes.io/docs/setup/production-environment/#production-worker-nodes))
- 리눅스에서 프로세스에 할당된 리소스를 제한하기 위해 control groups를 사용한다. kublet과 container runtime 모두 control group을 통해 po, container에 대한 리소스 관리를 수행하고 cpu/memory request, limit을 설정한다. control group을 사용하기 위해 kublet과 container runtime은 cgroup driver를 사용해야 한다. kublet과 container runtime이 동일한 cgroup driver를 사용하고 동일하게 구성되는 것이 중요하다. ([Container Runtimes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#cgroup-drivers))
- container runtime으로 containerd를 사용하는 경우 설정 파일에서 sandbox image를 변경할 수 있다. ([Container Runtimes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#override-pause-image-containerd))
- no당 최대 po 실행 갯수는 110개 ([Considerations for large clusters](https://kubernetes.io/docs/setup/best-practices/cluster-large/))
- no가 시작되면 각 no의 kubelet은 no object에 zone과 관련된 label(`topology.kubernetes.io/region`, `topology.kubernetes.io/zone`)을 추가한다. ([Running in multiple zones](https://kubernetes.io/docs/setup/best-practices/multiple-zones/#node-behavior))
- no가 k8s cluster의 no에 join하기 위한 요구 사항을 검증을 위해 node conformance test를 수행할 수 있다. ([Validate node setup](https://kubernetes.io/docs/setup/best-practices/node-conformance/))
- kube-proxy는 각 no에서 네트워크 규칙을 이용해 svc의 구현을 담당한다. ([Cluster Architecture](https://kubernetes.io/docs/concepts/architecture/#kube-proxy))

- no의 graceful/non-graceful shutdown 설정 고려 ([Node Shutdowns](https://kubernetes.io/docs/concepts/cluster-administration/node-shutdown/))
- no의 cgroup v2 설정 고려 ([About cgroup v2](https://kubernetes.io/docs/concepts/architecture/cgroups/))
- kubelet의 대부분 flag는 deprecated이며 대신 config file을 통해 설정하는 것을 권장한다. control plane 구성 요소의 경우 아직은 flag를 이용한 설정을 사용하는 것 같다.
- kubelet에서 container runtime에 접근하기 위한 endpoint 설정 고려 ([Container Runtime Interface (CRI)](https://kubernetes.io/docs/concepts/architecture/cri/#api))
- kubelet에서 image 용량에 따른 gc 설정 고려 ([Garbage Collection](https://kubernetes.io/docs/concepts/architecture/garbage-collection/#container-image-lifecycle))
- kubelet에서 image age에 따른 gc 설정 고려 ([Garbage Collection](https://kubernetes.io/docs/concepts/architecture/garbage-collection/#image-maximum-age-gc))
- kubelet이 특정 label을 마음대로 수정할 수 없도록 NodeRestriction admission plugin 설정 고려 ([Assigning Pods to Nodes](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#node-isolation-restriction))
- kubelet이 image credential provider을 사용해 동적으로 credential을 얻도록 사용 고려([Extending Kubernetes](https://kubernetes.io/docs/concepts/extend-kubernetes/#kubelet-image-credential-provider-plugins))
- kubelet의 authentication, authorization 설정 고려([Kubelet authentication/authorization](https://kubernetes.io/docs/reference/access-authn-authz/kubelet-authn-authz/))
- kubelet의 client certificate rotation은 `.rotateCertificates` 필드, server certificate 요청 및 rotation은 `.serverTLSBootstrap` 필드 사용([TLS bootstrapping](https://kubernetes.io/docs/reference/access-authn-authz/kubelet-tls-bootstrapping/#certificate-rotation))

### addon
- 구성 요소: `dns`, `web ui`, `container resource monitoring`, `cluster-level monitoring`
- addon의 기본 limit은 일반적으로 작은 크기의 cluster에서의 경험을 통해 수집한 데이터를 기반으로 하기 때문에 조정 필요 ([Considerations for large clusters](https://kubernetes.io/docs/setup/best-practices/cluster-large/#addon-resources))
- cluster의 사이즈가 커짐에 po의 resource 증량이 필요할 수 있으며, vpa의 recommender mode를 이용해 리소스의 제안 용량을 확인한다. ([Considerations for large clusters](https://kubernetes.io/docs/setup/best-practices/cluster-large/#addon-resources))

- k8s object의 status를 노출하는 kube-state-metrics 설치 고려 ([Metrics for Kubernetes Object States](https://kubernetes.io/docs/concepts/cluster-administration/kube-state-metrics/))

## 요약
- k8s의 autoscaling 옵션: pod hpa, vpa, cluster autoscaler, addon resizer
- linux container는 격리를 위해 namespace, cgroup(control group) 기술을 사용한다.
  - namespace:
    - `mnt`: mount points
    - `user`: user and group ids
    - `uts(unix time sharing)`: hostname, nis domain name
    - `pid`: process ids
    - `network`: network devices, stacks, ports, etc
    - `ipc(inter process communication)`: system V IPC, POSIX message queues
    - `control groups`: control group root directory
  - cgroup
    - 메모리, cpu, 네트워크 대역폭 등
- 이론적으로 container는 모든 리눅스 시스템에서 실행될 수 있지만 호스트의 커널을 사용하기 때문에 커널 버전에 영향을 받는다. 뿐만 아니라 특정 하드웨어 아키텍쳐용(x86, ARM 등)으로 만들어진 image는 해당 아키텍쳐에서만 실행될 수 있다.
- docker 기준 k8s는 po내 모든 container가 동일한 linux namespace를 공유하도록 docker를 설정한다. 파일 시스템의 경우 image에 저장되어 있어 기본적으로 완전히 분리된다.
  - 동일한 namespace: network, uts, ipc
  - 다른 namespace: uid, pid, mnt
- po의 manifest 중 spec.containers[\*].ports[\*].containerport는 container가 노출하는 포트 정보를 명시하기만 할 뿐 다른 기능은 없다. 이를 생략한다고 해서 포트를 통해 po에 연결할 수 있는 여부에 영향을 미치지 않는다.
- po는 k8s에서 배포할 수 있는 가장 작은 단위로, 이미 실행 중인 po 내에 container를 배포할 수 없다. 물론 po에 포함된 container가 po 내에서 재시작 될 수 있다.
- static po는 다른 po들(control plane이 담당)과 다르게 해당 no의 kubelet이 직접 관리한다. 그리고 static po에 대한 object를 API server에 게시한다(API server를 통해 object를 변경하는 것은 불가능). 즉 static po가 특정 kubelet에 종속되어 있지만 API server를 통해 조회가 가능하다. static po의 .spec에서는 다른 API object를 참조할 수 없다(예를 들어 sa, cm, secret 등).
- po 내 container는 app container, init container, sidecar container가 있다. app container는 `.spec.containers`, init container, sidecar container는 `.spec.initContainers` 필드를 사용해 정의한다. sidecar container는 `.spec.initContainers[].restartPolicy`가 Always인 init container다. `.spec.restartPolicy`는 app container, init container에만 영향을 미친다. `.spec.restartPolicy`가 OnFailure, Always일 경우 init container에게 모두 OnFailure로 적용된다.
- container 로그는 하루 단위, 10MB 크기 기준으로 롤링(rolling)된다.
- k8s namespace는 리소스 이름의 범위를 제공한다. 뿐만 아니라 리소스를 격리하는 것외에도 특정 사용자가 지정된 리소스에 접근할 수 있도록 허용하고, 개별 사용자가 사용할 수 있는 컴퓨팅 리소스를 제한하는 데에도 사용된다. 하지만 리소스간 격리는 제공하지 않는다. 즉 서로 다른 namepsace에 존재하는 리소스더라도 통신할 수 있다. 네트워크 격리는 k8s와 함께 배포되는 네트워킹 솔루션에 따라 다르다.
- container의 생명주기와 관련해 hook을 제공하며, handler를 구현함으로써 hook에 대한 이벤트를 처리할 수 있다.
  - PostStart: container와 비동기적으로 실행된다. 하지만 PreStart가 완료되지 않으면 container는 running state에 도달할 수 없다.
  - PreStop: container에 TERM 시그널이 전송되기 전에 실행된다. po의 terminationGracePeriodSeconds 설정보다 PreStop, TERM signal 전송 및 처리 시간이 더 오래걸이면 container는 비정상 종료 될 수도 있다. po가 terminated 상태가 되고 난 후, prestop hook -> container TERM sinal 전송된다.
- svc의 .spec.selctor는 .spec.type이 ClusterIP, NodePort, LoadBalancer일 떄만 적용됨. ExternalName일 때는 무시된다고 documentation에 명시되어 있지만, ep는 생성됨을 확인(물론 프록시는 되지 않음). 
- svc의 .spec.selector가 없는 svc는 일반적인 svc와 동일하게 동작한다. 하지만 ep를 자동으로 생성하지 않으며 사용자가 직접 생성/관리해야 하는 책임이 있다.
- svc의 .spec.type을 ExternalName으로 바꾸거나 반대로 바꾸지 않는 이상 svc가 수정되더라도 clusterIP는 고정 값이다.
- svc의 .spec.clusterIP는 .spec.type이 NodePort, ClusterIP, LoadBalancer일 때 자동 할당되거나 사용자가 설정할 수 있다.
- svc의 .spec.clusterIP가 None: headless svc를 의미(cluster ip 미할당). kube-proxy가 해당 svc를 다루지 않기 때문에 프록시, 로드밸런싱되지 않는다. 즉 .spec.ports[*]를 명시해도 의미가 없다.
  - .spec.selector가 있으면 svc DNS lookup 시 pod들의 ip 목록(A 레코드)가 조회됨. ep도 생성됨. .spec.clusterIP: "None"이 아니면 오류. 조회된 pod의 ip와 po의 노출 po로 접근할 수 있다.
  - .spec.selector가 없으면 svc DNS lookup 시 아무것도 조회 안됨
- svc의 .spec.type이 ClusterIP: \<cluster ip>:\<port>로 접근 가능
- svc의 .spec.type이 NodePort: \<node ip>: \<node port>, \<cluter ip>:\<port>로 접근 가능 
- svc의 .spec.type이 ExternalName: .spec.externalName 필드 필수로 설정 필요. .spec.clusterIP 핃드를 명시하지 않거나 값을 ""으로 설정 필요. cluster ip, port 할당되지 않음. svc DNS lookup 시 CNAME 레코드 조회됨.
- volume은 po의 수명주기와 같은 ephemeral, po의 수명주기와 상관없는 persistent volume을 사용할 수 있다. po 내 .spec.volumes[*] 필드에서는 persistent volume을 사용하기 위해 pvc를 사용해 pv resource를 요청한다. 즉, 직접 pv를 명시하는 것은 아니다. generic ephemeral pv의 경우에는 inline pvc를 사용하기 때문에 reclaim policy가 Retain이면 po의 수명주기와 상관없이 pv가 삭제되지 않는다. generic ephemeral pv의 경우 pvc의 이름이 po에 따라 다르게 생성되기 때문에 동일 static po에 대한 claim 시 1개 po의 pvc에 대해서만 bound 된다.
- 동일 pvc를 사용한다면 여러 po에서 동일 pv를 바인딩할 수 있다.
- 사용자는 pvc의 .spec.storageClassName 필드를 사용해 pv를 dynamic provisioning할 sc를 명시한다. 이 때 pvc는 먼저 요청에 매칭되는 static pv를 찾고 없으면 dynamic pv를 생성한다. 해당 필드가 ""라면 dynamic provisioning을 비활성화한다는 의미다. pv는 pvc, pvc는 po에 바운딩 되며 관련 finalizer를 갖는다. pv가 pvc로부터 release된 이후, reclaim policy에 따라 삭제 또는 보존될 수 있다. pvc내 .spec.volumeName, .spec.claimRef 필드를 통해 특정 pvc와 특정 pv를 바인딩할 수 있다.
- 1개의 pv는 1개의 pvc와 바운딩된다는 것을 명심해야 한다. 대신 1개의 pvc는 여러 po에서 사용할 수 있다.
- pv는 아래 state를 갖는다.
  - Available: claim에 대해 아직 바운드 되지 않은 리소스
  - Bounded: claim에 바운딩된 volume
  - Released: claim이 삭제되었지만, 리소스는 아직 cluster가 reclaim하지 않음
  - Failed: volume이 dynamic reclaim에 대해 실패함

---
## 명령어
- `kubectl get ${RESOURCE} -l`: label selector
- `kubectl get ${RESOURCE} -L`: 출력 column에 추가할 label
- `kubectl get ${RESOURCE} --field-selector`: field selector
- `kubectl get ${RESOURCE} --as ${USER} --as-group {GROUP}`: impersonation
- `kubectl auth whoami`: 사용자 인증에 대한 정보 확인(selfsubjectreview resource)
- `kubectl auth can-i`: 사용자의 동작에 대한 인가 정보 확인(selfsubjectaccessreview resource)
- `kubectl certificate (approve|deny)`: csr에 대한 승인/거부
- `kubectl delete ${RESOURCE}/${NAME} --cascade`: cascade deletion. 기본값은 background
- `kubectl drain ${NODE}`: API-initiated eviction
- `kuvectl apply --validate --dry-run`: client validation, dry run 수행
- `kubectl cluster-info`: control  plane의 주소(kube-apiserver)와 `kubernetes.io/cluster-service` label이 true인 service의 주소를 노출한다.
- `kubectl cluster-info dump --output-directory -A`: 클러스터의 전체 정보와 현재 ns, kube-system ns의 전체 정보를 stdout으로 출력한다.

---
## 체크리스트
- po내 ports[*].hostPort에 사용된 port는 호스트 netstat 조회 시 보이지 않음. 하지만 type=ClusterIP svc로 expose 시 netstat에 조회됨
- svc externalIPs 설정 시, no의 IP로 svc 접근 가능
- local pv의 경우, pv 생성 시 디렉토리가 호스트 내 존재해야 함. 자동 생성되지 않음 확인
- hpa의 cpu 사용률 계산 시 po를 선택하는 방법 알아보기
- container의 liveness, readiness, startup probe들도 back-off 재시작이 있는지

---
## k8s addon(plugin)
- aws EFS csi driver
  - elastic file system에 대해 access point를 생성하면 실제 file system의 root directory를 숨길 수 있다. 생성한 access point를 통해 접근하면 실제 file system의 루트 디렉토리가 아닌 access point의 루트 디렉토리에 접근한다.
  - access point를 생성할 때 설정 가능한 옵션은 다음과 같다.
    - root directory path: access point의 루트 디렉토리로 사용할 경로(file system 기준 절대 경로).
    - udi, gid, secondary gid: acccess point를 통해 접근하는 사용자의 uid, gid를 강제한다. 기본적으로 efs의 경우 root user(UID 0)만 rwx 권한을 갖는다. 다른 유저에 대해서는 직접 권한을 부여해야한다.
    - OwnerUid, OwnerGiD, Permissions: root directory path를 생성할 떄 사용할 디렉토리의 소유자, 그룹 권한 정보
  - ```
    EFS CSI driver supports dynamic provisioning and static provisioning. Currently Dynamic Provisioning creates an access point for each PV. This mean an AWS EFS file system has to be created manually on AWS first and should be provided as an input to the storage class parameter(parameters.fileSystemId). For static provisioning, AWS EFS file system needs to be created manually on AWS first. After that it can be mounted inside a container as a volume using the driver.
    ```
      - dynamic provisioning
        - AWS 상에 file system이 먼저 생성되어 있어야 한다.
        - pv에 따라 해당 file system 내에서 access point를 생성한다.
        - access point를 사용해 file system에 접근할 경우 동일 file system 내에서도 특정 경로를 투르 디렉토리로 인식하도록 만든다.
        - access point를 생성 시 설정 가능한 [파라미터](https://github.com/kubernetes-sigs/aws-efs-csi-driver/tree/master#storage-class-parameters-for-dynamic-provisioning)
        - access point의 root directory 경로는 \${.parameters.basePath}/\${.parameters.subPathPattern}가 된다.
      - static provisioning
        - AWS 상에 file system이 먼저 생성되어 있어야 한다.
        - access point 없이 file system에 접근이 가능하다.
        - pv를 생성할 때 [`.csi.volumeHandle`](https://github.com/kubernetes-sigs/aws-efs-csi-driver/blob/master/examples/kubernetes/access_points/README.md#efs-access-points) 필드를 통해 file system id, sub-path, access point 설정이 가능하다. 포맷은 `[FileSystemId]:[Subpath]:[AccessPointId]`
      - efs csi driver 설치 시, `delete-access-point-root-dir` 파라미터를 통해 dynamic provisioning을 통해 생성된 pv가 삭제될 떄 관련 access point의 삭제 여부도 제어할 수 있다.
  - efs csi driver에서 지원하는 accessmode는 소스코드 상 ReadWriteOnce(RWO), ReadWriteMany(RWX)를 지원하는 것으로 보인다(https://github.com/kubernetes-sigs/aws-efs-csi-driver/blob/7d0b76bca26f7e0c258f3cdc68286aacffbfe5b3/pkg/driver/node.go#LL36C3-L37C3).
    - efs csi driver의 경우 CSI ephemeral volumes은 지원하지 않는다.
- aws lbc(load balancer controller)
  - aws application elb의 경우 k8s ing resoucre, network elb의 경우 k8s svc resourcew를 통해 provision된다.
    - network elb(svc)
      - svc의 .spec.ports[*].port가 aws network elb 상에서 listener로 등록되며, 각 listener의 모든 target은 svc의 .spec.selector에 매칭되는 po의 집합이다. 각 listener의 target group은 k8s 상에서 TargetGroupBinding(crd)로 구현된다.
      - ing와 같이 ingress group을 사용해 여러 svc에서 1개의 aws network elb를 공유하는 것은 불가능하다. 단지 1개의 svc에서 정의된 .spec.ports[*] 필드를 사용해 여러 listener를 만들고 실제 target group은 해당 svc의 .spec.selector에 매칭되는 po들의 집합이다.
    - application elb(ing)
      - annotation을 통해 listener port를 지정할 수 있으며, 기본적으로 .spec.rules[*]에 설정된 rule 별로 설정된 svc가 listener의 target group으로 등록된다. target group은 k8s 상에서 TargetGroupBinding(CRD)로 구현된다.
      - annotation을 사용해 1개의 rule에 대한 target group에 대한 condition, action을 설정할 수 있다. 하지만 target group에 대한 health check, order, protocol, attributes 등은 ing에서 한 번만 지정되기 때문에 해당 설정을 공유하게 된다.
