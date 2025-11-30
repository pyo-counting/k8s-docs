## about docs
- v1.31 버전 기준 작성
- 3_Concepts/4_Workloads/1_Pods/2_Init Containers부터 정리 필요
- 실제 k8s의 인증서 생성과 목록에 대해 확인해보기

## Kubernetes
### general
- k8s의 모든 구성 요소를 k8s cluster 내에서 container로 관리(static pod)하는 것을 권장한다. 하지만 container 실행을 담당하는 kubelet은 container로 실행할 수 없다. ([Getting started](https://kubernetes.io/docs/setup/))
- k8s의 control plane은 linux에서 실행되도록 디자인됐다. 물론 cluster 내에서 애플리케이션은 windows를 포함한 다른 os에서 실행할 수도 있다. ([Getting started](https://kubernetes.io/docs/setup/#what-s-next))
- ns의 limit, hierarchical limit을 고려한다. ([Production environment](https://kubernetes.io/docs/setup/production-environment/#set-limits-on-workload-resources))
- k8s는 내부적으로 유저 인증 정보를 저장하지 않는다. 대부분의 웹 서비스나 인증 서버들은 사용자 정보를 내부적으로 저장하여 사용자로부터 인증 정보를 전달 받았을 때 저장된 정보를 바탕으로 인증을 처리한다(예를 들어 웹 사이트에서 계정과 비밀번호를 입력 받아 유저 DB를 조회하여 사용자 인증을 처리). k8s는 이와 다르게 따로 인증 정보를 저장하지 않고 외부 인증 시스템에서 제공해주는 신원 확인 기능들을 활용하여 사용자 인증을 하고 유저를 인식(identify)한다. 이러한 특징으로 인해 k8s에서는 쉽게 인증체계를 확장할 수 있다. k8s 내부 인증체계에 종속되는 부분이 거의 없기 때문이다. k8s는 사용자 인증체계를 전부 외부 시스템 (혹은 메커니즘)에 의존한다고 볼 수 있다. (X.509, HTTP Auth, Proxy Authentication 등) ([PKI certificates and requirements](https://kubernetes.io/docs/setup/best-practices/certificates/))
- k8s는 구성 요소간의 안전한 통신, 인증을 위해 PKI x.509 server certificate 인증서를 사용한다. 기본적으로 k8s general 목적, etcd, front-proxy(옵션)를 위한 ica가 필요하다. 이후 필요한 인증서 생성은 k8s에 위임할 수 있다(private key를 cluster에 복사하는 작업이 위험하다고 생각하면 필요한 인증서를 모두 직접 생성할 수도 있다). 뿐만 아니라 sa에서 사용하는 public, private key 쌍도 필요하다. ([PKI certificates and requirements](https://kubernetes.io/docs/setup/best-practices/certificates/#configure-certificates-manually))
- 사용자(kubeconfig 파일)를 위한 인증서는 kubernetes-ca로 인증 및 생성한다. ([PKI certificates and requirements](https://kubernetes.io/docs/setup/best-practices/certificates/#configure-certificates-for-user-accounts))
- k8s v1.25버전부터 kube-apiserver가 server side field validation을 제공한다. 이는 kubectl의 --validate flag와 동일한 기능을 수행한다. ([Objects In Kubernetes](https://kubernetes.io/docs/concepts/overview/working-with-objects/#server-side-field-validation))
- 각 object는 resource 유형 별로 고유한 name을 갖는다. 또한 모든 k8s object는 cluster lifecycle 동안 고유한 UID를 갖는다. resource 내에서 동일한 name을 갖는 object가 존재할 수 없다. 물론 해당 object를 삭제하고 동일한 이름을 갖는 새로운 object를 생성할 수 있다(이 때 UUID는 다름). name은 동일한 resource에 대해 모든 API 버전(`.apiVersion` 필드)에서 유일해야 한다. API resource는 API group, resource type, namespace(namespaced resource일 경우)로 구분된다. 즉, API 버전은 상관이 없다. ([Object Names and IDs](https://kubernetes.io/docs/concepts/overview/working-with-objects/names/#names))
- 사용자가 설정하는 name은 resource URL에서 `/api/v1/pods/<name>`와 같이 object를 참조하는 데 사용된다. 가장 보수적으로 RFC 1035를 준수하는 것이 편하다. resource 마다 object naming 규칙에 대한 제약 사항이 더 많을 수도 있다. ([Object Names and IDs](https://kubernetes.io/docs/concepts/overview/working-with-objects/names/#rfc-1035-label-names))
- label은 k8s에서 기본 중요 grouping으로 사용된다. label key는 /로 구분된 prefix(optional), name 형태로 구성된다. `kubernetes.io/`, `k8s.io/` 접두사는 k8s core system에서 사용되도록 예약됐다. ([Labels and Selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/#syntax-and-character-set))
- label selector는 2개의 타입(equality-based, set-based)을 지원(`kubectl get -l`)한다. ([Labels and Selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/#label-selectors))
- ns는 여러 사용자가 있는 환경에서 여러 resource의 분리를 위해 사용한다. ns와 quota를 resource 사용에 대한 정책을 관리할 수 있다. 운영 환경에서는 default ns를 사용하는 것을 권장하지 않는다. ([Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/#when-to-use-multiple-namespaces))
- k8s cluster에는 `default`, `kube-system`, `kube-node-lease`, `kube-public` 기본 ns가 존재한다. ns에 대해 `kube-` 접두사는 k8s 시스템을 위해 예약됐기 때문에 사용하지 않는다. ([Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/%2523working-with-namespaces))
- svc 생성 시 ns의 이름을 사용해 cluster 내에서 domain을 생성한다(포맷은 `<service-name>.<namespace-name>.svc.cluster.local`). TLD와 동일한 이름을 갖는 ns의 경우 public TLD를 덮어쓸 수 있기 때문에 위험하다. 그렇기 때문에 제한된 사용자만 ns를 만들 수 있도록 제한한다. ([Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/#namespaces-and-dns))
- k8s는 모든 ns에 immutable label인 `kubernetes.io/metadata.name`을 설정한다. ([Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/#automatic-labelling))
- annotation key는 /로 구분된 prefix(optional), name 형태로 구성된다. `kubernetes.io/`, `k8s.io/` 접두사는 k8s core system에서 사용되도록 예약됐다. ([Annotations](https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/#syntax-and-character-set))
- 각 resource의 필드 필터링을 위해 field selector를 사용(`kubectl get --field-selector`)할 수 있다. 모든 resource가 공통적으로 `metadata.name`, `metadata.namespace` 필드를 지원하며, resource 종류에 따라 지원하는 추가 필드가 다르다. ([Field Selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/field-selectors/))
- finalizer는 삭제 마킹된 resource를 완전히 삭제하기 전에 특정 조건이 충족될 때까지 대기하도록 k8s에 지시하는 namespaced key다. finalizer는 일반적으로 실행항 코드를 지정하지 않는다. 대신 일반적으로 annotation과 유사한 특정 resource에 대한 key 목록이다. finalizer가 존재하는 resource를 삭제할 동작은 다음과 같다. ([Finalizers](https://kubernetes.io/docs/concepts/overview/working-with-objects/finalizers/))
  - 해당 resource를 삭제하려고 할 때 kube-apiserver는 finalizer 필드 값을 확인하고 다음을 수행한다.
    - 삭제 요청 시간을 `.metadata.deletionTimestamp` 필드 값을 추가해 ojbect를 수정한다.
    - `metadata.finalizers` 필드가 빈 상태가 될때까지 object가 삭제되지 않도록 한다.
    - HTTP 202 status code(Accepted)를 반환한다.
  - finalizer를 관리하는 controller는 object 삭제가 요청됐음을 나타내는 `.metadata.deletionTimestamp` 필드 설정에 대한 업데이트를 확인한다.
  - controller는 해당 resource에 대해 finalizer의 요구 사항을 충족하려고한다. finalizer 조건이 만족될 때마다 controller는 resource의 finalizer 필드에서 해당 키를 삭제한다. finalizer 필드가 빈 값이 되면 deletionTimestamp가 설정된 object가 자동으로 삭제(gc)된다.
- label(`metadata.labels`), finalizer(`.metadata.finalizers`), owner reference(`metadata.ownerReferences`)는 비슷해 보이지만 각기 다른 목적으로 사용된다. 모두 객체간의 관계를 나타낸다는 공통점을 갖지만 label의 경우 controller가 관련있는 객체들에 대한 변화를 추적할 때 사용하는 반면, owner reference는 관련 객체를 삭제할 때 종속 관계를 파악하고 cascading deletion(background, foreground)를 수행하는 데 사용한다. 이 삭제 단계에서 k8s는 finalizer 필드도 같이 고려한다. ([Finalizers](https://kubernetes.io/docs/concepts/overview/working-with-objects/finalizers/#owners-labels-finalizers))
- owner reference는 resource 간 소유권 관게를 정의(예를 들어 rs와 po)하고 부모 resource 삭제 시 자식 resource를 삭제하는 생명주기 메커니즘을 갖는다. 이에 반해 finalizer는 resource의 의존성이나 사용 중인 상태를 표현하는 데 사용된다(예를 들어 pv와 po의 관계)(finalizer는 의존성, owner reference는 종속성을 나타낸다). ([ChatGPT]())
- cross-namespace owner reference는 허용되지 않는다. namespaced object는 cluster-scoped, namespaced owner를 가질 수 있다. 반면 cluster-scoped object는 cluster-scoped owner만 가질 수 있다. ([Owners and Dependents](https://kubernetes.io/docs/concepts/overview/working-with-objects/owners-dependents/#owner-references-in-object-specifications))
- 일부 툴에서 기본적으로 사용하는 `app.kubernetes.io` prefix label을 모든 resource마다 관리하는 것을 권장한다. ([Recommended Labels](https://kubernetes.io/docs/concepts/overview/working-with-objects/common-labels/))
- k8s control plane의 핵심은 kube-apiserver다. kube-apiserver는 HTTP API를 노출하여 최종 사용자, cluster의 다양한 부분, 외부 구성 요소가 서로 통신할 수 있도록한다. k8s API를 사용하면 k8s에서 API object의 state를 조회하고 수정할 수 있다(예: po, ns, cm 등). 대부분의 작업은 kubectl, kubeadm과 같은 CLI를 통해 수행된다. 이러한 도구들은 k8s API를 사용한다. 물론 REST call을 사용해 API에 직접 접근할 수도 있다. k8s는 k8s API를 사용하여 애플리케이션을 개발하려는 사람들을 위한 client libraries를 제공한다. ([The Kubernetes API](https://kubernetes.io/docs/concepts/overview/kubernetes-api/#openapi-interface-definition))
- k8s는 hub-and-spoke API 패턴을 사용한다. control plane 구성 요소 중 kube-apiserver만 HTTPS 443 listen port를 노출하며 no, po는 kube-apiserver와만 통신한다. kube-apiserver는 1개 이상의 authentication, authorization을 설정해야 한다. no는 cluster의 public root certificate가 provision 되어야 한다. 이를 통해 no는 유효한 client credential을 사용해 안전하게 kube-apiserver에 연결할 수 있다. kubelet에 제공되는 client credentials은 client certificate 형식이 권장된다. po는 sa를 사용해 kube-apiserver에 안전하게 연결할 수 있다. 이를 위해 k8s는 po가 생성될 때 public root certificate와 유효한 bearer token을 자동으로 주입한다. po는 인증서가 아닌 bearer token을 이용해 kube-apiserver에 인증한다. default ns의 kubernetes svc는 kube-apiserver의 HTTPS endpoint로 redirect(kube-proxy가 수행)되는 virtual ip로 구성되어 있다. ([Communication between Nodes and the Control Plane](https://kubernetes.io/docs/concepts/architecture/control-plane-node-communication/#node-to-control-plane))
- default ns의 kubernetes svc는 kube-apiserver의 HTTPS 엔드포인트로 redirect(kube-proxy가 수행)되는 virtual ip로 구성되어 있다. ([Communication between Nodes and the Control Plane](https://kubernetes.io/docs/concepts/architecture/control-plane-node-communication/#node-to-control-plane))
- k8s는 no의 heartbeat, control plane 구성 요소의 leader election, kube-apiserver의 신원 제공을 위해 lease 개념을 사용한다. lease는 k8s resource로 제공되기 때문에 사용자 workload에서도 사용할 수 있다. ([Leases](https://kubernetes.io/docs/concepts/architecture/leases/))
- 기본적으로 `.metadata.ownerReference.blockOwnerDeletion` 필드가 true일 경우 소유자 object를 삭제할 때 종속 object가 모두 삭제되지 않는 한 소유자 object를 삭제할 수 없다(즉, 종속 object 삭제 -> 소유자 object 삭제 수행이 필요). 하지만 `kubectl delete` 명령어를 사용할 경우 `--cascade` flag에 따라 무시될 수 있다. 기본 값은 background로 무시되며 foreground(기본 값은 background)일 경우 정상 동작한다. ([Garbage Collection](https://kubernetes.io/docs/concepts/architecture/garbage-collection/#cascading-deletion))
- API-initiated eviction는 eviction API를 사용해 graceful pod termination을 트리거하는 eviction object를 생성한다. API-initiated eviction는 po의 `PodDisruptionBudgets`, `terminationGracePeriodSeconds`을 존중한다. API를 사용해 po에 대한 eviction object를 생성하는 것은 po에 대한 delete 작업을 수행하는 것과 동일하다. ([API-initiated Eviction](https://kubernetes.io/docs/concepts/scheduling-eviction/api-eviction/))
---

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
- etcd는 cluster의 설정 정보를 저장하기 때문에, 가용성과 보안을 위해 별도의 서버에 구축 및 백업하는 절차를 갖는 것을 권장한다. cluster의 경우에는 더더욱 Event object 저장을 위한 별도의 etcd를 운영을 고려해야 한다. ([Production environment](https://kubernetes.io/docs/setup/production-environment/#production-control-plane), [Considerations for large clusters](https://kubernetes.io/docs/setup/best-practices/cluster-large/#etcd-storage))
- kube-scheduler의 NoVolumeZoneConflict 로직은 pv의 label, `.spec.nodeAffinity` 정보를 참조해 po가 위치할 az에 존재하는 no에 po를 할당한다. pv의 az 정보를 나타내기위해 label, `.spec.nodeAffinity`을 이용할 수 있으며 해당 정보를 추가하는 것은 cloud provider, storage provisioner에 따라 다르다. aws-ebs-csi-driver의 경우 label 대신 `.spec.nodeAffinity` 필드를 사용해 az 정보를 나타낸다. ([Running in multiple zones](https://kubernetes.io/docs/setup/best-practices/multiple-zones/#storage-access-for-zones))
- k8s v1.25버전부터 kube-apiserver가 server side field validation을 제공하며 1.27 버전부터 기본 활성화됐다. 이 기능은 kube-apiserver가 etcd에 리소스 manifest를 저장하기 전에 구조와 필드를 검증하는 기능이다. 과거에는 kubectl이 버전이 낮거나 비활성화 됐을 때 검증이 불가한 문제, kube-apiserver에서의 알 수 없는 필드 무시와 같은 문제로 클라이언트 측에서의 예상하지 못한 동작에 대한 트러블슈팅이 어려웠다. 이 기능을 통해 kube-apiserver는 알 수 없는 필드, 잘못된 데이터 타입, 변경 불가능한 필드에 대한 오류에 대해 오류를 응답한다. `kubectl --validate` 명령어를 통해 검증 수준을 제어할 수 있다(만약 kube-apiserver에 연결할 수 없는 경우 직접 kubectl이 검증을 수행). 이는 server side apply와는 다른 기능이다. server side apply는 kube-apiserver가 manifest의 소유권 관리 및 병합을 목적으로 한다. ([Objects In Kubernetes](https://kubernetes.io/docs/concepts/overview/working-with-objects/#server-side-field-validation))
- kube-apiserver: kube-apiserver -> kubelet 통신 시, 기본적으로 kube-apiserver는 kubelet의 server certificate를 검증하지 않는다. 검증을 위해 `--kubelet-certificate-authority` flag에 kubelet의 ca certificate를 설정해 kubelet으로의 연결을 안전하게 수행할 수 있다(대안으로 SSH tunneling 사용 가능). kube-apiserver -> kubelet 통신 경우는 다음과 같다. kube-apiserver가 kubelet에 no, po 상태에 대한 추가 정보 요청 등, po의 로그조회 (`kubectl logs`), 실행 중인 po에 대한 attach(`kubectl attach`, `kubectl exec`), 실행 중인 po에 대한 트래픽 포워딩(`kubectl port-forward`) ([Communication between Nodes and the Control Plane](https://kubernetes.io/docs/concepts/architecture/control-plane-node-communication/#control-plane-to-node))
- kube-apiserver: kube-apiserver -> no, po, svc와 직접 통신하는 경우는 kube-apiserver의 proxy 기능을 사용할 때다. proxy 기능은 kube-apiserver의 내장 기능으로 kube-apiserver를 통해 po, svc, no에 접근하는 경우에 사용된다. 대표적인 예시로 `kubectl cluster-info` 명령어를 사용해 조회되는 목록이다. 해당 목록에는 k8s kube-apiserver의 도메인과 `kubernetes.io/cluster-service` label이 true인 svc에 접근하기 위한 kube-apiserver의 주소를 출력한다. 물론 해당 목록에 조회되지 않더라도 kuber-apiserver의 API 명세서를 참고해 접근할 수 있다. ([Communication between Nodes and the Control Plane](https://kubernetes.io/docs/concepts/architecture/control-plane-node-communication/#api-server-to-nodes-pods-and-services))
- kube-controller-manager(node controller): node controller는 no의 생명 주기 동안 여러 작업을 수행한다. 첫 번째로 no가 등록될 때 CIDR 블락을 할당한다. 두 번째로 controller의 내부 no 목록을 cloud provider의 사용 가능한 시스템 목록을 참고해 최신 상태로 유지하는 것이다. 클라우드 환경에서 실행할 때 no가 unhealthy(Ready `.status.conditions[*].type`이 Unknown or False) 상태가 되면, node controller는 no에 대한 시스템이 이용 가능한지 cloud provider에 확인한다. 이용이 불가할 경우 node controller는 no 목록에서 해당 no를 삭제한다. 세 번째로 no의 상태를 모니터링한다. node controller는 다음과 같은 책임이 있다(kubelet의 [Node-pressure Eviction](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/), [API-initiated Eviction](https://kubernetes.io/docs/concepts/scheduling-eviction/api-eviction/)와 비교). ([Nodes](https://kubernetes.io/docs/concepts/architecture/nodes/#node-controller))
    - no가 unreachable 상태가 될 경우, no의 .status 필드의 Ready condition을 업데이트(node controller는 Ready condition을 `Unknown`으로 변경) 한다.
    - no가 unreachable(Unknown condition) 상태로 남아있는 경우, unreachable no에 있는 po를 eviction 하기 위해 `node.kubernetes.io/unreachable` taint key를 추가한다. k8s는 po에 명시적으로 `node.kubernetes.io/not-ready`, `node.kubernetes.io/unreachable` toleration key를 설정하지 않으면 해당 toleration과 `tolerationSeconds=300`을 설정하기 때문에 첫 eviction까지 5분을 기다린다.
      - eviction 상세 동작은 다음과 같다. 대부분의 경우 node controller는 초당 eviction 비율을 `--node-eviction-rate`(기본값 0.1)로 제한한다. 즉, 10초당 1개의 no에서만 po를 제거한다. 이 기본 동작은 cluster의 large cluster 여부, healthy 또는 unhealthy zone인지에 따라 달라진다.
      - no에 대한 eviction rate(`--node-eviction-rate`, `--secondary-node-eviction-rate`)는 `--unhealthy-zone-threshold`, `--large-cluster-size-threshold`에 따라 달라진다.
        - large cluster일 경우 `--unhealthy-zone-threshold`가 0.55 이상(unhealthy zone)이면 `--secondary-node-eviction-rate`을 사용한다.
        - small cluster일 경우 `--unhealthy-zone-threshold`가 0.55 이상(unhealthy zone)이면 `--secondary-node-eviction-rate` 값이 0 즉, eviction을 중지한다.
        - 특이 케이스는 다음과 같다. multi zone으로 구성된 경우 1개의 zone의 모든 no가 unhealthy가 되면 이 때는 `--node-eviction-rate`을 사용해 빠르게 pod eviction을 진행해 다른 zone으로 po를 옮기는 작업을 수행한다. 그리고 만약 모든 zone의 no가 unhealthy가 되면 이 때는 eviction을 아예 중단한다.
- kube-controller-manager(node controller): node와 관련된 flag는 다음과 같다.
  - `--service-cluster-ip-range`: 클러스터 내에서 svc에 할당할 ip cidr. `--cluster-cidr`와 겹치지 않아야 한다. `--allocate-node-cidrs`가 true여야 한다.
  - `--cluster-cidr`: 클러스터 전체 내 po에 할당할 ip cidr. `--allocate-node-cidrs=true`와 같이 사용해 각 no에 서브넷을 할당할 수 있도록 해야한다. 그리고 `--service-cluster-ip-range`와 겹치지 않아야 한다.
  - `--node-cidr-mask-size`: (기본값 24). no에 할당할 서브넷 마스크 크기. no는 해당 서브넷 내에서 po에 ip를 할당한다. 예를 들어, --cluster-cidr가 192.168.0.0/16이고 --node-cidr-mask-size가 24라면, 각 노드는 /24 크기의 CIDR 블록(예: 192.168.1.0/24)을 할당받습니다.
  - `--allocate-node-cidrs`: 각 no에 `--cluster-cidr`, `--node-cidr-mask-size` 기반 서브넷을 할당할지 여부
  - `--node-monitor-period`:(기본값 5s). kube-controller-manager가 kube-apiserver를 통해 no의 상태를 확인하는 주기
  - `--node-monitor-grace-period`: (기본값 40s) no를 unhealthy로 마킹하기 전에 대기하는 시간. 이 값은 kubelet의 `.nodeStatusUpdateFrequency`보다 충분히 큰 값이어야 한다.
  - `--node-startup-grace-period`: (기본값: 1m0s) starting no가 unhealthy로 마킹되는 것을 방지하기 위해 기다리는 시간
  - `--large-cluster-size-threshold`: (기본값: 50) large cluster에 대한 기준 값. 이는 eviction 로직에 영향을 준다. 만약 multiple zone이 구성된 경우 이 값은 각 zone 별로 적용된다.
  - `--unhealthy-zone-threshold`: (기본값: 0.55) unhealthy zone으로 판단하기 위한 unhealthy no(최소 3개)의 최소 비율.
  - `--node-eviction-rate`: (기본값: 0.1) zone이 healthy 상태일 때 po를 eviction하는 node의 rate.
  - `--secondary-node-eviction-rate`: (기본값: 0.01) zone이 unhealthy 상태일 때 po를 eviction하는 node의 rate. large cluster가 아닐경우 이 값은 0으로 간주된다.
- kube-scheduler는 no에 실행 중인 po에 대한 충분한 리소스가 있음을 보장한다. kube-scheduler는 no에 존재하는 container의 resource request에 대한 총합이 no의 capacity보다 크지 않음을 확인한다. request의 총합은 kubelet에 의해 관리되는 모든 container를 포함하며 container runtime을 통해 직접 실행된 container와 kubelet이 제어하지 않는 프로세스는 제외한다. ([Nodes](https://kubernetes.io/docs/concepts/architecture/nodes/#node-capacity))
- k8s에서 controller는 cluster의 상태를 관찰 한 다음 경우에 따라 생성 또는 변경을 요청하는 control loop다. 각 controller는 cluster의 current state를 desired state와 최대한 동일하게 만들기 위해 노력한다. controller는 적어도 1개의 k8s의 resource 타입을 추적한다. 이러한 object는 `.spec` 필드를 통해 desired state를 표현한다. 해당 resoure에 대한 controller는 object의 current state를 `.sepc`에 정의된 desired state에 가깝게 유지하기 위한 책임을 갖는다. 이를 위해 controller는 일반적으로 kube-apiserver를 통해 제어(예를 들어 kube-controller-manager의 job controller의 po 제어)하지만 controller 자체에서 직접 제어할 수도 있다. 외부 state와 상호 작용하는 controller는 kube-apiserver에서 desired state를 찾은 후, 외부 시스템과 직접 통신해 current state를 desired state에 가깝게 하기 위해 노력한다. 여기서 중요한 점은 controller가 desired state를 만들기 위해 약간의 변화를 만들고, current state를 cluster의 kube-apiserver에 보고한다는 것이다. cluster는 언제든지 작업이 발생하고 자동으로 실패를 고칠수 있다. 이는 잠재적으로 cluster가 안정적인 상태에 도달하지 못하는 것을 의미한다. cluster를 위한 controller가 실행 중이고 유용한 변경을 수행할 수 있는 한 전체 상태가 안정적인지 아닌지는 중요하지 않다. k8s는 기본적으로 control plane에 여러 내장 controller를 갖는 kube-controller-manager이 필요하다. ([Controllers](https://kubernetes.io/docs/concepts/architecture/controller/))
- control plane 구성 요소(예를 들어 kube-scheduler, kube-controller-manager)의 여러 replica는 kube-system ns에 저장된 lease object를 사용해 leader를 관리한다 ([Leases](https://kubernetes.io/docs/concepts/architecture/leases/#leader-election))
- kube-apiserver는 kube-system ns에 저장된 lease object를 통해 k8s 전체 시스템에 kube-apiserver에 대한 정보를 제공한다. 이 기능 자체만으로는 유용하지 않지만 클라이언트가 kube-apiserver 인스턴스 수를 확인하는 메커니즘에 사용될 수 있다. lease 이름에 사용되는 SHA256 hash는 각 kube-apiserver의 서버 hostname을 기반으로 한다. 각 kube-apiserver는 cluster 내에서 고유한 hostname을 사용하도록 구성되어야 한다. 동일한 hostname을 사용하는 새로운 kube-apiserver 인스턴스는 새로운 식별자를 사용해 기존의 lease를 덮어쓰며 새로운 lease object를 생성하지 않는다. kube-apiserver가 사용하는 hostname을 확인하기 위해 `kubernetes.io/hostname` label을 확인한다. ([Leases](https://kubernetes.io/docs/concepts/architecture/leases/#api-server-identity))
- cloud-controller-manager는 각 클라우드와 호환되는 제어 로직을 포함하는 k8s control plane 구성 요소다. cloud-controller-manager를 사용해 k8s cluster와 cloud provider의 API를 연결할 수 있으며 다른 구성 요소와 분리해서 관리할 수 있다. k8s와 cloud 인프라 간의 상호 운용성 로직을 분리함으로써 cloud-controller-manager 구성 요소는 cloud provider가 k8s 프로젝트와 별개로 기능을 릴리즈할 수 있다. cloud-controller-manager는 plugin mechanism을 사용해 서로 다른 cloud provider의 플랫폼을 k8s와 통합할 수 있도록 설계된다. cloud-controller-manager는 control plane의 구성 요소가 아닌 addon으로 실행할 수도 있다. cloud-controller-manager에 포함된 controller는 node controller, route controller, service controller가 있다. cloud-controller-manager는 k8s resource에 대한 다양한 권한이 필요하다.  ([Cloud Controller Manager](https://kubernetes.io/docs/concepts/architecture/cloud-controller/))
  - node controller: 클라우드 인프라의 서버 정보를 기반으로 k8s no object 정보를 업데이트하는 책임이 있다. 그리고 no의 상태를 확인하는 역할도 수행한다. no가 응답하지 않는 경우 controller는 cloud provider API를 사용해 서버의 상태(비활성/삭제/종료)를 확인한다. no가 cloud에서 삭제된 경우 controller는 해당 no 객체를 k8s cluster에서 삭제한다. 일부 cloud provider는 node controller, node lifecycle controller로 분리해서 구현하는 경우도 있다.
  - route controller: route controller는 k8s cluster의 다른 no에 있는 container가 서로 통신할 수 있도록 cloud 내에서 route를 구성한다. cloud provider에 따라 route controller는 po 네트워크를 위해 ip 주소 CIDR block을 할당한다.
  - service controller: svc는 cloud에서 제공하는 load balancer, ip 주소, network packet filtering, target health check와 같은 cloud 구성 요소와 통합된다. service controller는 해당 구성 요소가 필요한 svc object를 생성할 때 cloud provider API와 상호 작용해 load balancer와 같은 인프라 구성 요소를 구성한다.
- k8s는 resource 요청을 kube-apiserver가 다른 peer kube-apiserver로 proxy 처리할 수 있는 alpha 기능이 포함된다. 이 기능은 하나의 cluster에서 서로 다른 버전의 k8s에 대한 kube-apiserver가 있을 때(예를 들어, k8s의 새로운 release가 장기간 rollout 하는 동안) 유용하다. 이를 통해 cluster 관리자는 resource 요청(업그레이드 중에 수행)을 올바른 kube-apiserver로 direct함으로써 안전하게 업그레이드할 수 있는 가용성이 높은 cluster를 구성할 수 있다. 이 proxy를 사용하면 업그레이드 프로세스에서 발생할 수 있는 예기치 않은 404 Not Found 오류를 방지할 수 있다. 이 메커니즘을 mixed version proxy라고 부른다. ([Mixed Version Proxy](https://kubernetes.io/docs/concepts/architecture/mixed-version-proxy/))
- pss, psa는 psp를 대체하는 개념이다. pss는 k8s 내장 정책으로 level 별, k8s 버전 별 별도의 정책 목록을 정의한다. psa는 pss를 ns 단위에서 적용하기 위한 개념으로 내장 admission controller를 통해 구현된다. psa는 ns 단위로 정책을 위반할 때 어떤 동작을 수행할지에 대한 mode를 지원한다. ([Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/), [Pod Security Admission](https://kubernetes.io/docs/concepts/security/pod-security-admission/))
  - pss 정책 level은 Privileged -> Baseline -> Restricted 순서로 엄격하며 이전 정책 목록을 포함한다.
  - pss, psa는 cluster 레벨에서의 보안 정책 적용 관점으로, workload 또는 po `.spec`에서 적용되는 보안 사항을 감시한다.
  - psa의 audit, warning mode는 workload 생성 시에 정책 검사가 지원되지만 enforce mode는 po 생성 시에만 검사가 지원된다.
  - admission controller 설정을 통해 정책 검사에 대한 예외를 설정할 수 있다.
- kube-controller-manager 내 pod garbage collector(PodGC)는 terminated po(phase가 `Succeeded`, `Failed`)의 수가 --terminated-pod-gc-threshold(기본 값 12500)을 초과할 때 정리한다. 뿐만 아니라 아래 조건을 만족하는 po도 정리한다. PodGC는 po를 정리하는 과정에서, 아직 종료 단계(non-terminal phase)가 아닌 po를 Failed로 표시하기도 한다.
  - orphan po: 더 이상 존재하지 않은 no에 binding된 경우(orphan po를 정리할 때는 '파드 중단 조건(Pod disruption condition)'을 추가함)
  - 스케줄되지 않은 종료 중인 po
  - `node.kubernetes.io/out-of-service` taint가 있는 non-ready no에 있는 종료 중인 po
- k8s에서 scheduling은 kubelet이 po를 실행할 수 있도록 적절한 no를 찾는 것을 말한다. preemption(선점)은 cluster 내 resource가 부족한 상황에서 우선 순위가 높은 po가 no에 스케줄링 될 수 있도록 우선 순위가 낮은 po를 종료하는 것을 말한다. eviction은 no에서 po를 종료하는 것을 말한다. pod disruption은 no에서 po가 자발적 또는 비자발적으로 종료되는 프로세스를 말한다. voluntary disruption은 사용자에 의해 의도적으로 수행된다. involuntary disruption은 의도적이지 않은 것으로 no의 리소스 부족과 같은 불가피한 문제 또는 실수로 인한 삭제로 발생할 수 있다. ([Scheduling, Preemption and Eviction](https://kubernetes.io/docs/concepts/scheduling-eviction/))
- kube-scheduler는 k8s의 기본 scheduler이며 control plane의 구성 요소로 실행된다. kube-scheduler는 scheduling framework라는 plugin 아키텍처를 갖는다. kube-scheduler는 기본적으로 여러 plugin을 제공하며 추가적으로 사용자는 plugin을 개발하고 직접 kube-scheduler를 컴파일해서 사용할 수 있다. scheduling framework는 몇 가지 extension point을 정의한다. 스케줄러 plugin은 하나 이상의 extension point에서 호출되도록 등록한다. 이러한 plugin 중 일부는 스케줄링 결정을 변경할 수 있고 일부는 정보 제공에 불과하다. 하나의 po를 스케줄링하기 위한 프로세스는 2개의 phase(scheduling cycle, binding cycle)로 구성된다. scheduling cycle은 po를 위한 no를 선택하고 binding cycle은 해당 결정을 cluster에 적용한다. 이 두 cycle을 "scheduling context"라고 부른다. scheduling cyle은 내부적으로 크게 filtering, scoring 단계로 수행된다. kube-scheduler는 새로 생성되거나 아직 스케줄링 되지 않은(unscheduled) po를 실행할 최적의 no를 선택한다. po 또는 container가 요구 사항을 가질 수 있기 때문에 scheduler는 po의 특정 스케줄링 요구 사항을 충족하지 않는 no를 걸러낸다(filtering). 물론 po를 생성할 때 no를 지정할 수도 있지만 이는 일반적이지 않으며 특수한 경우에만 사용한다. cluster에서 po의 스케줄링 요구 사항을 충족(filtering 단계)하는 no를 feasible no라고 한다. feasible no가 없는 경우 scheduler가 po를 배치할 수 있을 때까지 po는 unscheduled 상태로 유지된다. scheduler는 po에 대한 feasible no를 모두 찾은 다음, 각 feasible no에 대해 일련의 함수를 실행해 점수를 계산하고(scoring), 가장 높은 점수를 받은 feasible no를 선택해 po를 할당한다. 동일한 점수를 가진 no가 여러 개인 경우 kube-scheduler는 이들 중 하나를 무작위로 선택한다. 그리고 kube-scheduler는 binding cycle을 통해 이러한 결정을 kube-apiserver에 알린다. k8s 1.7 이전에는 k8s binding 내장 리소스를 사용했지만 1.7에서 deprecated 됐으며 대신 po의 binding subresource를 사용한다. binding subresource를 통해 po와 no다 연결(binding)되고, 이로 인해 po의 `.spec.nodeName` 필드에 해당 no의 이름이 설정된다. ([Kubernetes Scheduler](https://kubernetes.io/docs/concepts/scheduling-eviction/kube-scheduler/#kube-scheduler), [Scheduling Framework](https://kubernetes.io/docs/concepts/scheduling-eviction/scheduling-framework/))
- kube-scheduler의 설정 파일에서 `.profiles` 필드를 이용해 각기 다른 scheduling 설정을 갖는 프로파일을 설정할 수 있다. 각 profile은 `.profiles[*].schedulerName` 필드의 이름을 갖고 po의 `.spec.schedulerName` 필드에서 참조함으로써 해당 po의 담당 scheduler를 설정할 수 있다. 기본적으로 default-scheduler profile만 생성된다. ([Kubernetes Scheduler](https://kubernetes.io/docs/concepts/scheduling-eviction/kube-scheduler/#kube-scheduler-implementation))
- kube-scheduler의 NodeAffinity plugin을 사용해 cluster에 기본 node affinity를 적용할 수 있다. ([Assigning Pods to Nodes](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#node-affinity-per-scheduling-profile))
- kube-scheduler의 PodTopologySpread plugin을 사용해 사용자가 cluster에 기본 pod spread constraints를 적용할 수 있다. kube-scheduler의 기본 topology constraints는 다음과 같다. plugin 설정을 변경해 기본 topology constraints도 비활성화 할 수 있다. ([Pod Topology Spread Constraints](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/#cluster-level-default-constraints))
  ``` yaml
  defaultConstraints:
  - maxSkew: 3
    topologyKey: "kubernetes.io/hostname"
    whenUnsatisfiable: ScheduleAnyway
  - maxSkew: 5
    topologyKey: "topology.kubernetes.io/zone"
    whenUnsatisfiable: ScheduleAnyway
  ```
- kube-controller-manager(node controller)는 특정 조건이 만족될 때 po eviction을 위해 자동으로 no에 taint를 추가한다. no가 drain 되어야 하는 경우 node controller 또는 kubelet은 NoExecute effect를 `node.kubernetes.io/not-ready`, `node.kubernetes.io/unreachable` taint에 추가한다. 장애 상태가 정상으로 돌아오면 kubelet 또는 no controller가 관련 taint를 제거한다. no에 unreachable이면 kube-apiserver가 no의 kubelet과 통신에 실패할 수 있다. 그렇기 때문에 po에 대한 삭제를 kubelet에 통보할 수 없다. 그렇기 때문에 해당 no에 po는 계속해서 실행 중일 수 있다. 아래는 내장 taint 목록이다. ([Taints and Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/#taint-based-evictions))
  - `node.kubernetes.io/not-ready`: no가 준비되지 않았다. 이는 NodeCondition Ready가 "False"로 됨에 해당한다.
  - `node.kubernetes.io/unreachable`: no가 no controller에서 도달할 수 없다. 이는 NodeCondition Ready가 "Unknown"로 됨에 해당한다.
  - `node.kubernetes.io/memory-pressure`: no에 memory pressure이 있다.
  - `node.kubernetes.io/disk-pressure`: no에 disk pressure이 있다.
  - `node.kubernetes.io/pid-pressure`: no에 PID pressure이 있다.
  - `node.kubernetes.io/network-unavailable`: no의 네트워크를 사용할 수 없다.
  - `node.kubernetes.io/unschedulable`: no를 스케줄할 수 없다(예를 들어 no의 `.spec.unschedulable` 필드가 false일 때).
  - `node.cloudprovider.kubernetes.io/uninitialized`: kubelet의 "external" cloud provider와 같이 실행되는 경우 사용 불가능한 no로 표기하기 위해 taint를 추가한다. 이후, cloud-controller-manager의 controller가 이 no를 초기화하면 kubelet은 taint를 제거한다.
- kube-controller-manager(node controller)는 po의 eviction 뿐만 아니라 no의 condition에 따라 NoSchedule effect taint를 추가한다. 예를 들어 no가 node-pressure에 있는 경우 `node.kubernetes.io/disk-pressure` taint를 추가해 새로운 po가 scheduling되지 않도록 한다. ([Taints and Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/#taint-nodes-by-condition))
- kube-scheduler는 스케줄링 결정을 내릴 때 no condition을 확인하는 것이 아니라 taint를 확인한다. 이렇게 하면 no condition이 스케줄링에 직접적인 영향을 주지 않는다. ([Taints and Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/#taint-nodes-by-condition))
- pc, preemption과 관련된 kube-scheduler의 특징 및 동작은 다음과 같다. ([Pod Priority and Preemption](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/#effect-of-pod-priority-on-scheduling-order))
  - po에 대한 우선 순위가 활성화된 경우 scheduler는 scheduling queue에서 pending po의 우선 순위에 따라(우선 순위가 높은 pending po가 앞에 위치) 정렬를 수행한다. 결과적으로 요구 사항을 만족한다면 우선 순위가 높은 po가 먼저 스케줄링 된다. 만약 스케줄링에 실패할 경우 우선 순위가 낮은 po를 스케줄링하기 위해 시도한다.
  - po가 생성되면 먼저 스케줄링이 되기 위해 scheduling queue에 대기한다. scheduler는 qeueue에 있는 pending po를 선택해 no에 스케줄링하기 위해 노력한다. 만약 pending po의 모든 요구 사항을 만족하는 no가 없다면 preemption 로직이 트리거된다. pending po를 P라고 칭하자. preemption 로직은 우선 순위가 P보다 낮은 하나 이상의 po를 제거하면 P를 스케줄링할 수 있는 no를 찾으려고 시도한다. 만약 이러한 no가 발견되면, 우선 순위가 낮은 하나 이상의 po가 해당 노드에서 축출(evicted)된다. po가 제거된 후, P는 해당 no에 스케줄링될 수 있다.
  - po P가 no N에서 하나 이상의 po를 선점(preempt)할 때, po P의 `.status.nominatedNodeName` 필드 값은 no N의 이름으로 설정된다. 이 필드는 scheduler가 po P를 위해 예약된 리소스를 추적하는 데 도움을 주며, 사용자에게 cluster에서 발생한 preempt에 대한 정보를 제공한다. 하지만 po P가 반드시 "nominated Node"에 스케줄링되는 것은 아니다. victim po가 preempt된 후 설정에 따라 graceful termination period를 갖는다. scheduler는 victim po의 종료를 기다리는 동안 다른 no에 스케줄링이 가능한 경우, nominated no가 아닌 다른 no에 스케줄링할 수도 있다. 결과적으로 po의 `.status.nominatedNodeName`와 `.spec.nodeName` 필드 값이 다를 수 있다. 그리고 scheduler나 no N에서 po를 preempt 하더라도 po P보다 우선 순위가 더 높은 다른 po가 있을 경우 po P가 아닌 우선 순위가 더 높은 po를 할당할 수도 있다. 이런 경우 scheduler는 po P에 설정됐던 `.status.nominatedNodeName`을 다시 빈 값으로 설정한다. 이를 통해 scheduler는 po P가 다른 no에서 po를 preempt할 수 있도록 허용한다.
  - victim po가 preempt될 때 설정에 따라 graceful termination period를 갖는다. 이로 인해 preempt한 시점과 preempt를 야기한 우선 순위가 높은 po가 실제 해당 no에 스케줄링되기까지 시간 차이가 발생한다(물론 scheduler는 계속해서 다른 pending po에 대한 스케줄링을 수행한다). graceful termination period가 만료되고 victim po가 종료되면 해당 po를 스케줄링하기 위해 다시 scheduling queue에 넣는다. 결과적으로 victim po의 preempt 시점과 po가 스케줄링 되는 시점 간 시간 차이가 발생한다. 이 시간 차이를 최소화하기 위해 graceful termination period을 짧은 시간으로 설정할 수 있다.
  - pdb는 voluntary disruption에 대해 동시에 종료될 수 있는 po의 replica 개수 또는 비율을 제한한다. k8s는 po preempt에 대해 pdb를 지원하지만 완전 보장하지는 못하며 최선을 다한다(best effort). scheduler는 기본적으로 preempt에 의해 pdb를 보장할 수 있는 po를 찾기위해 노력한다. 하지만 찾지 못할 경우 pdb를 위반하더라도 우선 순위가 낮은 po를 삭제할 수 있다.
  - 각 no에 대해 po preemption을 고려할 떄 해당 no에서 우선 순위가 낮은 po를 preempt함으로써 우선 순위가 높은 pending po가 스케줄링 가능한지 확인한다. pending po가 no의 우선 순위가 낮은 po와 inter-pod affinity가 있는 경우 scheduler는 해당 no에서 어떠한 po도 preemption하지 않는다. 이 문제를 해결하기 위해 권장하는 방법은 inter-pod affinity를 사용할 때 우선 순위가 같거나 더 높은 po에 대해서만 고려한다.
  - no N이 po P의 스케줄링을 위해 preemption을 고려되고 있다고 가정한다. 다른 모든 조건은 만족하지만 no N에 po P가 스케줄링 되기 위해 다른 no의 po가 preemption 되어야 할 수도 있다(예를 들어 zone을 toepology key로 사용하는 pod anti-affinity일 경우). scheduler는 기본적으로 다른 no에 대한 preemption 기능은 없기 때문에 결과적으로 no N에 po P가 스케줄링 될 수없다.
  - preemption을 위한 여러 no가 있는 경우 scheduler는 우선 순위가 가장 낮은 po 집합을 가진 po를 선택하려고 시도한다. 하지만 이러한 po들이 preemption될 경우 pdb가 위반될 가능성이 있다면 scheduler는 더 높은 우선 순위의 po를 가진 다른 no를 선택할 수 있다. scheduler의 preemption 로직은 preemption 대상 선택 시 QoS를 고려하지 않는다.
- admission controller는 object를 생성(create), 삭제(delete) 또는 수정(modify)하는 요청에 적용된다. 또한 admission controller는 kube-apiserver proxy를 통해 po에 연결하는 요청과 같은 사용자 정의 동사(custom verbs)도 차단할 수 있다. admission controller는 object를 읽는(get, watch 또는 list) 요청은 차단하지 않는다(차단할 수 없다). 왜냐하면 읽기 작업은 admission control 계층을 거치지 않기 때문이다. admission control 메커니즘은 validating(검증) 또는 mutating(변경) 또는 둘 다일 수 있다. mutating controller는 수정되는 resource의 데이터를 변경할 수 있지만, validating controller는 그럴 수 없다. mutating -> validating admission controller 순서대로 실행된다. 일부 admission controller는 두 가지 역할을 모두 수행한다는 점에 유의해야 한다. 어느 단계에서든 admission controller 중 하나라도 요청을 reject하면 전체 요청은 즉시 reject되고 최종 사용자에게 오류가 반환된다. ([Admission Control](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#what-are-they))
- admission controller는 요청 object를 변경하는 것 뿐만 아니라 때때로 관련된 resource를 변경하는 side effect를 가질 수 있다. 예를 들어 quota 사용량이 대표적인 예시다. ([Admission Control](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/))
  1. 사용자가 cpu 1, 메모리 1GiB를 사용하는 po를 생성 요청을 수행한다.
  2. quota를 담당하는 admission controller에서 quota 사용량을 확인하고 충분한 경우 현재 사용량을 늘린다(side effect).
  3. 이후 admission controller에서 예를 들어, 보안 규정에 어긋나 reject를 수행한다. 결과적으로 quota 사용량은 증가했지만 po는 생성되지 않은 상태가 되며 복구(reclamation) 또는 조정(reconciliation) 프로세스가 필요하다. 왜냐하면 특정 admission controller는 해당 요청이 다른 모든 admission controller를 통과할 것이라고 확신할 수 없기 때문이다.
- kube-apiserver에는 여러 admission controller가 포함되어 있다. 뿐만 아니라 사용자를 위한 MutatingAdmissionWebhook, ValidatingAdmissionWebhook, ValidatingAdmissionPolicy admission controller가 있다. 앞의 두 webhook admission controller는 각각 설정된 mutating, validating admission control webhook을 실행한다. ValidatingAdmissionPolicy는 외부 HTTP 호출에 의존하지 않고 API 내에 선언적 검증 코드(CEL, common Expression Language)를 포함할 수 있는 방법을 제공한다. 이 세 가지 admission controller를 사용하여 admission 시점에 cluster 동작을 사용자가 커스텀할 수 있다. ([Admission Control](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#admission-control-phases))
- kube-apiserver의 `--enable-admission-plugins`, `--disable-admission-plugins` flag를 사용해 admission controller를 활성화, 비활성화할 수 있다. `--enable-admission-plugins`보다 `--disable-admission-plugins`가 우선 순위가 높다. 권장되는 admission controller는 기본적으로 활성화되어 있으므로([목록](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/#options)), 명시적으로 지정할 필요가 없다. MutatingAdmissionWebhook, ValidatingAdmissionWebhook을 비활성화하는 경우, `--runtime-config` flag를 사용해 API도 비활성화해야 한다.  ([Admission Control](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/))
---

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
    - sa token authentication mode 활성화는 kube-apiserver에 `--service-account-key-file` flag를 사용한다. kube-controller-manager에는 `--service-account-private-key-file` flag를 통해 개인키 파일을 설정한다. ([Authenticating](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#service-account-tokens), [PKI certificates and requirements](https://kubernetes.io/docs/setup/best-practices/certificates/#certificate-paths))
    - oidc token, webhook token, authentication proxy authentication mode 활성화는 [Authenticating](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#openid-connect-tokens) 참고
    - kubectl은 webhook, oidc provider로부터 사용자 credential 획득을 위해 client-go credential plugin을 사용할 수 있다.
    - eks의 경우 내부적으로 aws-iam-authenticator oss를 webhook token authenticator로 사용한다.
    - anonymous request authentication mode 활성화는 `--anonymous-auth` flag를 사용한다. ([Authenticating](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#anonymous-requests))
  - authorization mode 활성화는 `--authorization-mode` flag를 사용한다. 나열된 순서대로 authorization을 진행한다. ([Authorization](https://kubernetes.io/docs/reference/access-authn-authz/authorization/#authorization-modules))
    - node authorizor에 의해 인증되기 위해 kubelet은 system `system:node:<nodeName>` 사용자, `system:nodes` 그룹에 대한 credentials을 사용해야 한다. 해당 사용자, 그룹 형식은 [kubelet TLS bootstrapping](https://kubernetes.io/docs/reference/access-authn-authz/kubelet-tls-bootstrapping/)을 통해 생성된 kubelet의 identity와 일치한다. ([Using Node Authorization](https://kubernetes.io/docs/reference/access-authn-authz/node/))

### data plane (node)
- 구성 요소: `kubelet`, `container runtime`, `kube-proxy`
- 워크로드 scaling 시 coredns의 scaling도 고려가 필요하다. ([Production environment](https://kubernetes.io/docs/setup/production-environment/#production-worker-nodes))
- no 모니터링을 위한 node problem detector 설치 고려 ([Production environment](https://kubernetes.io/docs/setup/production-environment/#production-worker-nodes))
- 기본적으로 linux kernel은 IPv4 패킷이 network interface 간 라우팅되는 것을 허용하지 않는다. 대부분의 k8s cluster netwokring 구현은 필요한 경우 이 설정을 변경하지만 이외의 경우에는 관리자가 직접 변경해야 한다. ([Container Runtimes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#network-configuration))
- 리눅스에서 프로세스에 할당된 리소스를 제한하기 위해 control groups를 사용한다. kubelet과 container runtime 모두 control group을 통해 po, container에 대한 리소스 관리를 수행하고 cpu/memory request, limit을 설정한다. control group을 사용하기 위해 kubelet과 container runtime은 cgroup driver를 사용해야 한다. kubelet과 container runtime이 동일한 cgroup driver를 사용하고 동일하게 구성되는 것이 중요하다. cgroup driver는 cgroup을 관리하는 모듈을 의미한다. cgroup driver는 cgroupfs driver와 systemd driver가 존재한다. cgroupfs driver는 자신이 직접 cgroupfs을 통해서 cgroup을 제어한다. 반면 systemd driver는 systemd를 통해서 cgroup을 제어한다. ([Container Runtimes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#cgroup-drivers))
  - cgroupfs는 kubelet의 기본 cgroup driver다. cgroupfs를 사용하면 kubelet, container runtime은 직접 cgroup 파일시스템에 접근해 cgroup을 구성한다. systemd가 init system인 경우 cgroupfs driver를 사용하지 않는 것을 권장한다. 왜냐하면 systemd는 해당 시스템에서 자신만이 유일한 cgroup manager로 예상하기 때문이다. 추가적으로 [cgroup v2](https://kubernetes.io/docs/concepts/architecture/cgroups)를 사용하면 cgroupfs 대신 systemd를 사용해야 한다. 리눅스 distribution에서 systemd가 init system으로 사용되면, init 프로세스는 root control group(cgroup)을 생성 및 사용하면서 동시에 관리자 역할을 수행한다. systemd는 cgroup과 통합되어 있으며 systemd 당 cgroup을 할당한다. container runtime과 kubelet이 cgroupfs를 사용하도록 설정할 수 있다. systemd과 함께 cgroupfs를 사용하면 두 개의 다른 cgroup 관리자가 있다는 것을 의미한다. 단일 cgroup 관리자는 할당된 리소스 뷰가 단순하며, 사용 중인 리소스와 사용 가능한 리소스를 보다 일관되게 볼 수 있다. 시스템에 cgroup 관리자가 둘일 경우 해당 리소스에 대한 두가지 뷰가 있게된다. kubelet과 docker에 cgroupfs를 사용하도록 설정한 경우 리소스 pressure에 대한 불안정한 현상이 있다는 사람들의 보고가 있다. 이러한 불안정성을 완화하기 위해 systemd가 init systemd일 경우 container runtime, kubelet이 cgroup driver로 systemd를 사용하도록 설정을 변경하면 시스템이 안정화된다.
- container runtime에 따라 설정 파일에서 기본 sandbox (pause) image를 변경할 수 있다. ([Container Runtimes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#override-pause-image-containerd))
- no당 최대 po 실행 갯수는 110개 ([Considerations for large clusters](https://kubernetes.io/docs/setup/best-practices/cluster-large/))
- no가 시작되면 각 no의 kubelet은 no object에 zone과 관련된 label(`topology.kubernetes.io/region`, `topology.kubernetes.io/zone` 등)을 추가한다. ([Running in multiple zones](https://kubernetes.io/docs/setup/best-practices/multiple-zones/#node-behavior), [Node Labels Populated By The Kubelet](https://kubernetes.io/docs/reference/node/node-labels/))
- no가 k8s cluster의 no에 join하기 위한 요구 사항을 검증을 위해 node conformance test를 수행할 수 있다. ([Validate node setup](https://kubernetes.io/docs/setup/best-practices/node-conformance/))
- kube-proxy는 각 no에서 네트워크 규칙을 이용해 svc의 구현을 담당(userspace, iptables, ipvs mode를 지원)한다. ([Cluster Architecture](https://kubernetes.io/docs/concepts/architecture/#kube-proxy))
- kubelet의 대부분 flag는 deprecated이며 대신 config file을 통해 설정하는 것을 권장한다. control plane 구성 요소의 경우 아직은 flag를 이용한 설정을 사용하는 것 같다. ([kubelet](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/))
- kube-apiserver에 no를 등록하는 방법은 관리자가 no object를 생성하기(`.registerNode`가 false), kubelet이 kube-apiserver에 자신을 등록하기(`registerNode`가 true) 2가지다. 만약 no가 healthy 상태라면(즉, 필요한 모든 서비스가 실행 중) po를 실행할 자격이 있다. healthy 상태가 아니라면 healthy 상태가 되기 전까지 해당 no는 cluster와 관련된 행동에서 제외된다. k8s는 유효하지 않은 no의 object를 보존하면서 healthy 상태가 될때까지 게속 체크한다. health check를 멈추기 위해 no object를 직접 또는 controller가 삭제해야 한다. no를 교체하거나 업데이트해야 하는 경우 기존 no object를 먼저 kube-apiserver에서 제거하고 업데이투 후 다시 추가해야 한다. ([Nodes](https://kubernetes.io/docs/concepts/architecture/nodes/#management))
- k8s no가 보내는 heartbeat는 cluster가 각 no의 가용성을 파악하고 failure가 감지되면 조치를 취할 수 있도록 도와준다. 2가지 형태의 heartbeat(no object의 `.status` 필드 업데이트, `kube-node-lease` ns의 lease obejct 업데이트)가 있다. heartbeat와 관련된 kubelet의 설정은 다음과 같다. ([Nodes](https://kubernetes.io/docs/concepts/architecture/nodes/#self-registration-of-nodes))
  - `.nodeStatusUpdateFrequency`: (기본값 10s) kubelet이 no의 상태를 확인(변화가 있는지)하는 주기(kubelet은 no의 변화가 감지되면 no object의 `.status` 필드를 업데이트를 수행). 만약 lease 기능이 활성화되지 않았을 때는 실제 no object의 `.status` 필드 업데이트까지 수행한다. 이 경우 kube-controller-manager의 `--node-monitor-grace-period` flag 값을 고려해야 한다.
  - `.nodeStatusReportFrequency`: (기본값 5m) no의 상태 변화가 없을 경우 kubelet이 no object의 `.status` 필드를 업데이트(강제 full sync)하는 주기. lease 기능이 활성화 됐을 때만 유효한 설정이다(비활성화 시에는 이미 `.nodeStatusUpdateFrequency` 설정에 따라 `.status` 필드를 업데이트하기 때문). 기본 값은 5m이지만 이전 버전과의 호환성을 위해 `.nodeStatusUpdateFrequency` 가 명시적으로 설정된 경우 해당 값과 동일한 값으로 설정된다(이는 lease 개념이 없어 `.nodeStatusReportFrequency` 설정만 있던 이전 버전에서 사용자가 `.nodeStatusReportFrequency`를 설정하는 경우 실제 `.status`가 해당 설정 값 주기로 업데이트되는 것에 대한 호환성을 유지하기 위함).
  - `.nodeLeaseDurationSeconds`: (기본값 40) kubelet이 no의 lease object `.spec.renewTime`을 통해 no의 상태를 업데이트하는 주기. 해당 설정 값은 실제 시간을 나타내지 않으며 기본 값 40은 10s를 나타낸다. lease 업데이트가 실패하면 kubelet은 200ms를 시작으로 최대 7s까지의 지수 함수 backoff를 사용해 재시도를 수행한다.
- kubelet이 직접 cluster에 자신을 등록할 때 no의 `.status.capacity` 정보를 제공한다. 하지만 관리자가 no object를 직접 생성하는 경우 해당 정보를 직접 제공해야 한다. ([Nodes](https://kubernetes.io/docs/concepts/architecture/nodes/#node-capacity))
- k8s 구성 요소가 아닌 예약된 resource를 명시하기 위해 kubelet의 `.systemReserved` 설정을 사용한다. ([Nodes](https://kubernetes.io/docs/concepts/architecture/nodes/#node-capacity))
- kubelet의 TopologyManager [feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)를 활성화한 경우 kubelet은 리소스 할당 결정을 할 때 topology 힌트를 이용할 수 있다. ([Nodes](https://kubernetes.io/docs/concepts/architecture/nodes/#node-topology))
- no에 swap을 활성화하기 위해 kubelet의 `NodeSwap` feature gate 활성화(기본 값 true), kubelet의 `.failSwapOn`이 false(기본 값 true)어야한다. po가 swap을 사용하기 위해서는 kubelet의 `.swapBehavior`이 NoSwap (기본 값)이면 안된다. ([Nodes](https://kubernetes.io/docs/concepts/architecture/nodes/#swap-memory))
- cgroup v2 환경에서 k8s의 일부 기능([MemoryQoS](https://kubernetes.io/blog/2021/11/26/qos-memory-resources/))은 향상된 리소스 관리, 격리를 위해 cgroup v2를 전용으로 사용한다. ([About cgroup v2](https://kubernetes.io/docs/concepts/architecture/cgroups/))
- container runtime interface(CRI)는 kubelet과 container runtime 사이의 프로토콜이다. kubelet은 gRPC를 사용해 container runtime에 연결할 때 client로 동작한다. runtime endpoint, image service endpoint는 container runtime에서 사용 가능해야 하며 kubelet에서는 `.containerRuntimeEndpoint`, `.imageServiceEndpoint` 설정을 사용해 각각 설정할 수 있다. `.imageServiceEndpoint`를 명시하지 않으면 기본적으로 `.containerRuntimeEndpoint`을 사용해 image service에 연결한다. `.imageServiceEndpoint`는 container runtime service와 image service가 분리된 환경에서 사용한다(docker, containerd는 기본적으로 container runtime에서 image 관리와 container 실행 서비스를 모두 제공). 예를 들어 container runtime service와 image service가 분리된 환경에서 사용한다. ([Container Runtime Interface (CRI)](https://kubernetes.io/docs/concepts/architecture/cri/#api))
  - kubelet은 CRI를 통해 container runtime에게 container 생성을 지시하고, container runtime은 CNI를 호출해 container의 네트워크를 설정한다. ([Gemini]())
    - kubelet: 각 worker no에서 실행되며, kube-apiserver와 통신한다. 특정 no에 po를 생성하라는 명령을 받으면, 해당 po가 정상적으로 동작하도록 모든 작업을 총괄한다.
    - container runtime: container image 다운로드, container 생성 및 실행, 종료 등 container의 생명주기를 직접 관리한다.
    - cri: kubelet과 container runtime 사이의 통신을 위한 표준 인터페이스다(gRPC 기반). kubelet은 cri 덕분에 containerd를 쓰든 CRI-O를 쓰든 상관없이 동일한 명령으로 container 생성을 요청할 수 있다.
    - cni: container에 network namespace를 연결하고 IP 주소를 할당하는 등의 네트워크 설정을 전문적으로 처리하는 표준 인터페이스 및 플러그인이다. Calico, Flannel, Weave Net 등 다양한 CNI 플러그인이 있다.
      1. 명령 수신: kubelet은 kube-apiserver로부터 자신의 no에 새로운 po를 생성하라는 명세(Pod Spec)를 전달받는다.
      2. container 생성 요청 (kubelet → cri → container runtime)
          - kubelet은 po를 실행하기 위해 cri를 통해 container runtime에게 "이 image로 container를 만들어줘"라고 요청한다.
          - 이 요청에는 container image 정보, 볼륨, 환경 변수 등 필요한 모든 정보가 포함된다.
      3. container 생성 및 network namespace 준비
          - container runtime은 요청에 따라 container image를 다운로드하고, container를 실행할 기본적인 환경(샌드박스)과 독립된 네트워크 공간(network namespace)을 만든다.
          - 이 시점의 container는 아직 ip 주소가 없어 외부와 통신할 수 없다.
      4. 네트워크 설정 요청 (container runtime → cni)
          - container runtime은 만들어진 container의 network namespace 정보를 담아 설정된 cni plugin을 호출한다. "이 container에 네트워크를 연결해줘"라고 요청하는 것과 같다.
      5. 네트워크 설정 수행 (cni)
          - 호출된 cni plugin(예: Calico)은 다음 작업을 수행한다.
            - 가상 이더넷 인터페이스(veth pair) 생성
            - 한쪽 끝은 container의 network namespace에, 다른 쪽 끝은 host의 네트워크 브릿지에 연결
            - 파드 CIDR 대역에서 IP 주소를 할당하고 라우팅 규칙 설정
      6. 상태 보고: po가 ip를 할당받고 정상적으로 실행되면, container runtime은 이 사실을 kubelet에게 알리고, kubelet은 po의 최종 상태를 kube-apiserver에 보고한다.
- kubelet은 사용되지 않는 image에 대한 gc를 5분, 사용되지 않는 container에 대한 gc는 1분 마다 수행한다. 외부 gc 도구는 kubelet의 행동을 방해하고 필요한 container를 삭제할 수 있으므로 사용을 피해야 한다. `.imageGCHighThresholdPercent` 값을 초과한 디스크 사용량은 마지막으로 사용된 시간을 기준으로 오래된 image 순서대로 삭제하는 gc를 트리거한다. kubelet은 디스크 사용량이 `.imageGCHighThresholdPercent` 값에 도달할 때까지 image를 삭제한다. alpha 기능으로 디스크 사용량과 무관하게 로컬에 있는 사용되지 않는 image의 최대 시간을 설정할 수 있다. 이 기능을 사용하기 위해 kubelet의 `.ImageMaximumGCAge` feature gate를 활성화하고 kubelet 설정 파일에서 `.ImageMaximumGCAge` 필드를 사용하면 된다(kubelet이 재시작되면 계산 중이던 age는 초기화된다). container gc 기능의 경우 deprecated 됐으며([#127157](https://github.com/kubernetes/kubernetes/issues/127157#issuecomment-2333512962)) 대신 eviction(`.evictionHard`, `.evictionSoft`)을 사용한다.([Garbage Collection](https://kubernetes.io/docs/concepts/architecture/garbage-collection/#container-image-lifecycle))
- k8s는 기본 container runtime을 사용한다. 하지만 여러 container runtime을 사용하는 경우 po의 `.spec.runtimeClassName`을 이용해 cluster에 존재하는 [RuntimeClass](https://kubernetes.io/docs/concepts/containers/runtime-class/) object를 명시해 특정 container runtime을 사용할 수도 있다. ([Containers](https://kubernetes.io/docs/concepts/containers/))
- container image는 registry hostname(옵션), image name, tag 또는 digest 구성된다. digest는 image 내용을 hashing 알고리즘 적용한 hash 값으로 immutable이다. tag는 명시하지 않은 경우 latest 값으로 사용한다. po가 항상 동일한 container image 버전을 사용하는 것을 보장하기 위해 `<image-name>:<tag>` 대신 `<image-name>@<digest>`를 사용할 수 있다. ([Images](https://kubernetes.io/docs/concepts/containers/images/))
- `.spec.containers[*].imagePullPolicy`을 생략할 경우의 기본 동작은 다음과 같다. ([Images](https://kubernetes.io/docs/concepts/containers/images/#imagepullpolicy-defaulting))
  - tag 대신 digest를 설정한 경우 IfNotPresent 값으로 설정
  - tag가 :latest일 때 Always 값으로 설정
  - tag를 명시하지 않을 경우 Always 값으로 설정
  - :latest가 아닌 tag일 때 IfNotPresent 값으로 설정
- 기본적으로 kubelet은 한 번에 한 개의 image pull만 수행한다. `.serializeImagePulls`, `.maxParallelImagePulls` 필드를 사용해 병렬 image pull을 설정할 수 있다. 이 때 contaienr runtime의 image service가 병렬 image pull을 다룰 수 있는지도 확인해야 한다. 병렬 image pull이 활성화 됐더라도 kubelet은 1개의 po에 대해 여러 image를 병렬로 pull하지 않는다. 하지만 2개의 po가 서로 다른 image를 사용하는 경우에는 병렬로 image를 pull한다. ([Images](https://kubernetes.io/docs/concepts/containers/images/#serial-and-parallel-image-pulls))
- kubelet이 container runtime을 사용해 po의 container 생성을 시작할 때, `ImagePullBackOff`로 인해 container가 Waiting 상태에 있을 수 있다(po의 `.status.phase`가 Waiting). k8s는 back-off 딜레이를 통해 image pulling을 계속 시도한다. k8s는 시간 간격을 늘리면서 재시도를 수행하며 k8s에 코딩된 최대 시간 5분까지 시도한다. ([Images](https://kubernetes.io/docs/concepts/containers/images/#imagepullbackoff))
- container의 hostname은 po의 이름으로 설정된다. ([Container Environment](https://kubernetes.io/docs/concepts/containers/container-environment/))
- RuntimeClass resource에 설정 가능한 목록은 CRI(Container Runtime Interface)의 구현에 따라 다르다. 기본적으로 필수 설정해야 하는 필드는 `.handler`다. 이는 container runtime을 참조하는데 사용되며 각 container runtime마다 handler에 대응되는 설정을 갖는다. 만약 no마다 지원하는 container runtime이 다를 경우 RuntimeClass에 `.scheduling` 필드를 사용해 container runtime을 지원하는 no를 명시해야 한다. 그리고 po가 시스템의 리소스를 더 사용하는 경우를 대비해 `.overhead` 필드를 사용해 추가적인 리소스를 지정할 수도 있다. ([Runtime Class](https://kubernetes.io/docs/concepts/containers/runtime-class/))
- k8s는 container에 2개(`PostStart`, `PreStop`) lifecycle hook을 제공한다. hook을 통해 container는 lifecycle의 event를 인지할 수 있으며 이에 대응하는 lifecycle hook이 실행될 때 handler를 통해 구현한 코드를 실행할 수 있다. hook hanadler는 `Exec`, `HTTP`, `Sleep` 3가지 유형을 제공한다. httpGet, tcpSocket, sleep은 kubelet 프로세스에 의해 실행되며 exec는 container 내부에서 실행된다. PostStart 또는 PreStop hook이 실패하면 container를 kill한다. ([Container Lifecycle Hooks](https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/))
  - `PostStart`: container가 생성될 때 container의 ENTRYPOINT와 `PostStart` hook은 동시에 트리거된다. 하지만 hook 실행이 너무 오래 걸리거나 hang에 걸린다면 container는 running 상태에 도달할 수 없다.
  - `PreStop`: API 요청(예를 들어 `kubectl delete`)이나 liveness/startup probe 실패, preemption, resource contention 등의 management event로 인해 container가 종료되기 전에 트리거된다(container는 Terminating 상태가 된다). container가 이미 terminated 또는 completed 상태인 경우에는 PreStop hook 요청이 실패하며, hook은 container를 중지하기 위한 TERM(SIGTERM) signal이 보내지기 이전에 완료되어야 한다. po의 `terminationGracePeriodSeconds`는 PreStop hook이 실행되기 전에 시작된다. 예를 들어 terminationGracePeriodSeconds가 60이고 hook가 실행을 완료하는 데 55초가 걸리고 signal을 수신하고 정상적으로 종료하는 데 10초가 걸리면, container는 정상적으로 중지되기 전에 SIGKILL을 수신해 container가 강제 종료될 것이다(terminationGracePeriodSeconds가 총 시간 55+10보다 짧기 때문).
- hook handler의 로그는 po event에 노출되지 않는다. handler가 어떤 이유로 실패한다면 event를 브로드캐스트한다. PostStart의 경우 `FailedPostStartHook` event, PreStop의 경우 `FailedPreStopHook` event이다. ([Container Lifecycle Hooks](https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/#debugging-hook-handlers))
- po는 k8s에서 배포할 수 있는 가장 작은 단위로, 이미 실행 중인 po 내에 container를 배포할 수 없다. 물론 po에 포함된 container가 po 내에서 재시작 될 수 있다. po 내 container를 재시작하는 것과 po를 재시작하는 것과 혼동하면 안된다 po는 프로세스가 아니며 container를 구동하기 위한 환경이다. po는 삭제되기 전까지 지속된다. ([Pods](https://kubernetes.io/docs/concepts/workloads/pods/#working-with-pods))
- `.spec.os.name`은 kube-scheduler가 po를 스케줄링 할 no의 os를 결정하는 데 사용되는 필드가 아니며 pss를 위해 사용된다. ([Pods](https://kubernetes.io/docs/concepts/workloads/pods/#pod-os))
- 각 po에는 고유한 ip가 할당된다. po의 모든 container는 네트워크 ip주소, port를 포함하는 네트워크 namespace를 공유한다. po에 속한 container는 서로 localhost를 이용해 통신할 수 있다. container가 po 외부와 통신할 때 공유 네트워크 리소스를 어떻게 이용할지 조정해야한다. 또한 po 내 container끼리 SystemV semaphores, POSIX shared memory와 같은 표준 IPC를 이용해 통신할 수 있다. container가 다른 po의 container와 통신하기 위해서는 ip를 이용한 통신만 가능하다(OS 수준의 IPC를 이용하기 위해서는 설정 필요). ([Pods](https://kubernetes.io/docs/concepts/workloads/pods/#pod-networking))
- static po는 kube-apiserver의 관찰 없이 특정 no의 kubelet daemon에 의해 관리된다. deploy와 같이 대부분의 po는 control plane에 의해 관리되는 반면 static po는 kubelet이 직접 관리한다. static po는 항상 특정 node의 kubelet에 한정된다. static po의 주된 용도는 자체 호스팅 control plane을 실행하는 것이다: 즉, kubelet을 사용해 개별 control plane 구성 요소를 감독한다. kubelet은 각 static po에 대해 kube-apiserver에 mirror po(kubelet에 의해 관리되는 static po를 추적하는 object)를 자동으로 생성한다. 이는 no에 실행되는 po가 kube-apiserver에서 볼 수 있음을 의미하지만 kube-apiserver를 통해 제어는 하지 못한다. static po의 `.spec`에서는 다른 API object를 참조할 수 없다(예를 들어 sa, cm, secret 등). ([Pods](https://kubernetes.io/docs/concepts/workloads/pods/#static-pods))
- 기본적으로 활성화되어 있는 SidecarContainers feature gate는 init container에 `restartPolicy: Always`를 지정할 수 있도록 허용한다. Always 재시작 정책을 설정하면, 해당 container는 po의 전체 수명 주기 동안 계속 실행되는 sidecar로 취급된다. 이렇게 sidecar로 명시적으로 정의된 container는 메인 애플리케이션 container보다 먼저 시작하며, po가 종료될 때까지 계속 실행 상태를 유지된다. ([Pods](https://kubernetes.io/docs/concepts/workloads/pods/#how-pods-manage-multiple-containers))
- po의 상태는 `.status.phase`(Pending, Running, Succeeded, Failed, Unknown)를 통해 확인할 수 있다. 그리고 po의 상태를 더 세분화해서 각 항목 별로 성공/실패 여부를 알려주는 `.status.conditions[*]`()를 제공한다. 뿐만 아니라 사용자 정의 condition(readiness gate)를 `.spec.readinessGates[*]`필드를 사용해 추가할 수 있다.container의 상태는 `.state.containerStatuses[*].state`(Waiting, Running, Terminated)를 통해 확인할 수 있다. ([Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-phase))
  - po의 모든 container는 Ready 상태가 되었지만(readinessProbe 통과), readinessGates에 지정된 커스텀 조건 중 하나라도 False이거나 아직 설정되지 않았다면 kubelet은 po의 전체 상태는 "Not Ready"로 두지만, Pod의 conditions 필드에 ContainersReady라는 특별한 상태를 True로 설정한다. 이는 "po 내부의 애플리케이션들은 준비되었지만, 외부 의존성과 관련된 준비 조건이 아직 충족되지 않았다"는 것을 명확히 알려주는 유용한 신호다.
- k8s는 po `.spec.restartPolicy`을 사용하여 po 내 container의 장애를 관리한다. 이 정책은 오류나 다른 이유로 container가 종료될 때 k8s가 어떻게 반응할지를 결정하며, 다음과 같은 순서로 진행한다. ([Pods](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-restarts))
  1. Initial crash: container에 문제가 발생하면, k8s는 po의 `.spec.restartPolicy`에 따라 즉시 재시작을 시도한다.
  2. Repeated crashes: Initial crash 이후에도 container가 계속해서 실패하면, k8s는 후속 재시작에 대해 exponential back-off 지연을 적용한다. 이는 시스템에 과부하를 주는 무분별하고 반복적인 재시작 시도를 방지한다.
  3. CrashLoopBackOff state: 이 상태는 현재 exponential back-off 메커니즘이 특정 container에 적용되고 있음을 나타낸다. 해당 container는 반복적으로 실패하고 재시작하는 '크래시 루프(crash loop)'에 빠져 있다.
  4. Backoff reset: 만약 container가 특정 시간(10분) 동안 성공적으로 실행되면, k8s는 exponential back-off 지연 시간을 초기화하고, 이후에 발생하는 새로운 크래시는 첫 번째 크래시로 간주한다.
- `.spec.restartPolicy`는 po 내 애플리케이션 container, init container에 적용된다. sidecar container는 po 레벨의 restartPolicy 필드를 무시한다. sidecar container는 init container 중 `.spec.initContainers[*].restartPolicy`(해당 필드 init container에만 사용 가능하며 값은 Always만 사용 가능) 가 Always인 container다. init container가 성공 종료하지 못한다면 po의 `.spec.restartPolicy`가 OnFailure, Always일 경우에만 재실행한다. ([Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#restart-policy))
- container를 종료시키는 데 사용되는 stop signal은 container image 내 STOPSIGNAL 명령어를 통해 정의할 수 있다. 만약 image에 stop signal이 정의되어 있지 않다면 container를 종료시키기 위해 container runtime의 기본 신호(containerd와 CRI-O 모두 SIGTERM)가 사용된다. ([Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination-stop-signals))
- po의 termination flow는 다음과 같다. ([Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination))
  1. kubectl을 사용해 po를 삭제한다(grace period(`.spec.terminationGracePeriodSeconds`)의 기본 값은 30s).
  2. kube-apiserver 내에서 po는 grace period와 함께 "dead"로 간주되는 시간으로 업데이트된다(grace period의 countdown이 시작된다). kubectl describe로 확인 시 po는 "Terminating"으로 표시된다. po가 실행되는 no의 kubelet은 해당 po가 terminating으로 표시된 것을 확인하고 po의 종료 프로세스를 시작한다.
      1. container가 preStop hook을 설정한 경우, kubelet은 container 내부에서 hook을 실행한다. grace period가 만료된 후 preStop hook이 계속 실행 중이라면, kubelet은 2초의 작은 일회성 grace period 연장을 요청한다.
      2. preStop hook 실행이 완료된 후, kubelet은 container runtime을 트리거해 각 container 내부 1번 프로세스에 TERM signal을 전송한다.
  3. kubelet이 graceful shutdown을 실행하는 것과 동시에 control plane은 svc의 ep에서 해당 po를 제거한다. rs과 같은 workload resoucre는 더 이상 해당 po를 유효하다고 판단하지 않는다.
  4. kubelet은 po가 완전히 종료되는 것을 보장하기 위해 다음과 같은 동작을 수행한다.
      1. grace period가 완료 됐음에도 실행 중인 container가 있는 경우 forcible shutdown을 트리거한다. container runtime은 container내 실행 중인 모든 프로세스에 SIGKILL signal을 전송한다. 또한 kubelet은 pause container가 있는 container runtime에 대해 해당 container도 정리한다.
      2. po를 terminal phase(`Failed` 또는 `Succeeded`)로 변경한다.
      3. kubelet은 grace period를 0로 변경함으로써 po를 force deletion한다.
      4. kube-apiserver는 po object를 삭제한다.
- force deletion(예를 들어 `kubectl delete --force --grace-period=0`)이 수행되면, kube-apiserver는 no에서 po가 종료되었다는 kubelet의 확인을 기다리지 않는다. kube-apiserver에서 즉시 po를 제거하므로 동일한 이름으로 새로운 po를 생성할 수 있다. 즉시 종료되도록 설정된 po는 no에서 강제 종료되기 전에 짧은 grace period가 제공된다. 이러한 삭제는 실행 중인 po를 즉시 종료 및 삭제하지만 실제로 종료가 됐는지 여부는 확인하지 않는다. 그렇기 때문에 해당 po가 no에 계속 남아있을 수도 있게 된다. ([Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination-forced))
- po에 하나 이상의 sidecar container(Always 재시작 정책을 가진 init container)가 포함된 경우, kubelet은 마지막 메인 container가 완전히 종료될 때까지 sidecar container에 TERM 신호를 보내는 것을 지연한다. sidecar container는 po spec에 정의된 역순으로 종료된다. ([Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#termination-with-sidecars))
- po는 누군가에 의해 삭제되지 않는한 사라지지 않는다. 물론 불가피한 하드웨어 또는 시스템 소프트웨어 오류가 있을 수 있으며 이러한 상황을 involuntary disruption(비자발적 중단)이라고 부른다(예를 들어 node pressure eviction). 이와 반대로 사용자의 동작으로 인한 상황을 voluntary disruption(자발적 중단)이라고 부른다. k8s는 자발적 중단이 자주 발생하는 경우에도 고가용성 애플리케이션을 실행하는 데 도움이 되는 pdb를 제공한다. 모든 자발적 중단이 pdb에 연관되는 것은 아니다. 예를 들어 deploy, po에 대한 삭제는 pdb를 무시한다. pdb는 자발적 중단으로 일시에 중지되는 replica의 갯수를 제한하는 정책을 갖는다. 비자발적 중단은 pdb로는 막을 수 없지만 budget은 차감된다. 이러한 불가피한 상황을 애플리케이션의 비자발적 중단(involuntary disruption)이라고 부른다. deploy, sts의 rolling upgrade는 pdb의 제한을 받지 않는다. ([Disruptions](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/))
- pdb의 `.spec.maxUnavailable`, `.spec.minAvailable`은 갯수 또는 퍼센트 값을 가질 수 있다. 퍼센트 값일 경우 소수점 값을 올림처리해서 계산한다. pdb에는 두 필드 중 1개만 사용해야 한다. ([Disruptions](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/#pod-disruption-conditions))
- kubelet은 svc, po에 대한 cluster 내부 DNS record르 생성한다. kubelet은 pod의 `/etc/resolv.conf`을 설정해 pod가 cluster 내부 DNS record를 조회할 수 있도록 설정한다. 아래는 예시다.
  ``` sh
  nameserver 10.32.0.10
  search <namespace>.svc.cluster.local svc.cluster.local cluster.local
  options ndots:5
  ```
- k8s에서 po의 스케줄링을 제어하기 위한 설정은 다음과 같다.
 ([Assigning Pods to Nodes](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/))
  - nodeSelector: (po의 `.spec.nodeSelector`) no에 대한 label selector로 po가 스케줄링 no를 제한한다. 해당 label을 갖는 no가 없으면 po는 스케줄링되지 않는다.
    - kubelet이 `node-restriction.kubernetes.io/` 접두사를 갖는 label을 업데이트하지 못하도록 함으로써 특정 po에 대한 격리를 수행할 수 있다(po의 `.spec.nodeSelector`). 해당 label을 사용하기 위한 몇 가지 필요 사항이 있다. [Node authorizer](https://kubernetes.io/docs/reference/access-authn-authz/node/), NodeRestriction admission plugin 활성화가 필요하다.
  - affinity / anti-affinity: (po의 `.spec.affinity`) label selector에 비해 더 표현적이며, no 뿐만 아니라 po에 대한 affinity도 제공한다. 그리고 preferred 규칙을 사용해 scheduler는 매칭되는 no를 찾지 못할 경우에도 po를 스케줄링할 수 있다. IgnoredDuringExecution의 의미는 스케줄링이 완료된 후에는 조건을 보장하지 않는다는 것을 의미한다.
    - node affinity: (po의 `.spec.affinity.nodeAffinity`): matchExpressions의 operator를 NotIn, DoesNotExist를 사용해 node anti-affinity처럼 사용할 수 있다. `.spec.affinity.nodeAffinity.requiredDuringSchedulingIgnoredDuringExecution`, `.spec.affinity.nodeAffinity.preferredDuringSchedulingIgnoredDuringExecution`는 각각 hard, soft 정책을 나타낸다.
      - nodeAffinity의 soft, hard 정책을 같이 사용하면 두 조건을 모두(AND) 만족해야 한다.
      - nodeAffinity의 soft 정책을 여러개 사용할 경우 no별로 각 정책의 weight를 더해 가장 점수가 높은 no가 우선 순위를 갖는다.
      - nodeSelector, nodeAffinity를 모두 사용한다면 스케줄링이 되기 위해 두 조건을 모두(AND) 만족해야 한다.
      - nodeAffinity의 nodeSelectorTerms을 여러개 사용할 경우 명시된 nodeSelectorTerms 중 하나(OR)를 만족하는 no에도 po가 스케줄링 될 수 있다.
      - nodeSelectorTerms의 matchExpressions를 여러개 사용하는 경우 모든(AND) matchExpressions를 만족하는 no에만 po가 스케줄링 될 수 있다.
    - inter-pod affinity, pod anti-affinity: (po의 `.spec.affinity.podAffinity`, `.spec.affinity.podAntiAffinity`) 보통 po가 동일 po에 접근해야 하는 경우 사용한다. topology에 매칭되는 no 중 이미 실행 중인 다른 po의 label을 기반으로 po를 스케줄링한다. topologyKey는 반드시 명시해야한다. requiredDuringSchedulingIgnoredDuringExecution, preferredDuringSchedulingIgnoredDuringExecution는 각각 hard, soft 정책을 나타낸다. requiredDuringSchedulingIgnoredDuringExecution pod anti-affinity 규칙의 경우 LimitPodHardAntiAffinityTopology admission controller는 topologyKey를 `kubernetes.io/hostname`으로 제한한다. 다른 topology를 사용하고 싶다면 admission controller를 수정하거나 비활성화할 수 있다. affinity의 대상이 되는 po는 namespaces, namespaceSelector와 labelSelector, matchLabelKeys, mismatchLabelKeys 필드를 통해 지정할 수 있다. inter-pod affinity와 anti-affinity에는 상당한 양의 프로세싱이 필요하기에 대규모 cluster에서는 스케줄링 속도가 크게 느려질 수 있다. 수백 개의 no를 넘어가는 cluster에서 이를 사용하는 것은 추천하지 않는다. no에 일관된 label을 지정해야 한다. 즉, cluster의 모든 no는 topologyKey와 매칭되는 적절한 label을 가지고 있어야 한다. 일부 또는 모든 no에 topologyKey로 명시한 label이 없는 경우에는 의도하지 않은 동작이 발생할 수 있다.
  - nodeName: (po의 `.spec.nodeName`): 해당 affinity, nodeSelector보다 더 직접적인 no 선택 방법이다. scheduler는 해당 필드를 사용하는 po를 무시하며, 바로 해당 no의 kubelet이 해당 po를 자기 no에 배치하려고 시도한다. 이는 다른 규칙보다 우선 적용된다. 이는 advanced use case를 위한 목적으로만 사용하는 것을 권장한다. 빈 값일 경우 kube-scheduler의 스케줄링 대상 po로 인식되며, kube-scheduler는 스케줄링을 위한 no를 결정한 경우 po의 binding subresource를 통해 해당 필드를 no의 이름으로 설정되도록 한다.
  - pod topology constraints: (po의 `.spec.topologySpreadConstraints[*]`) po을 특정 topology(zone, az, node) 내에서 균등하게 분산하기 위해 사용한다. affinity는 no 또는 po와의 관계를 통해 po가 배치 될 no를 결정하기 위해 사용한다. 두 개념은 같이 사용할 수 있으며 pod topology constraints 내에서 node affinity, taint를 무시할지 여부도 정책으로 사용할 수 있다. 여러 constraints를 사용하는 경우 모두 만족하는 no를 고려한다. 기본적으로 동일 ns의 po만 고려한다. 이미 po가 배치된 no에 topology label이 삭제되면 해당 추가 po를 스케줄링할 때 해당 no의 po는 maxSkew 계산에 사용하지 않는다. po가 no에서 삭제되는 상황에도 constraints가 만족되는 것을 보장하지 않는다. 예를 들어 deploy의 scale down으로 po의 분포가 불균등할 수 있다. 이 경우를 대비해 [Descedhuler](https://github.com/kubernetes-sigs/descheduler)를 사용할 수 있다. tainted no에 존재하는 po도 계산된다. scheduler는 해당 시점에 cluster에 존재하는 no만 고려한다. 각 필드의 의미는 다음과 같다.
    - `topologyKey`: (required) no의 label key. 해당 label key를 갖고 동일한 값을 갖는 no는 동일 topology로 간주된다. 각 topology instance(no의 label key&value가 같은 집합)를 domain이라고 부른다. 그리고 domain을 구성하는 no가 `nodeAffinityPolicy`, `nodeTaintsPolicy` 요구 사항을 충족(요구 사항이 없는 경우에도)하는 경우 eligible domain이라고 한다. kube-scheduler는 각 domain에 po를 균등하게 배포하려고 한다. 
    - `minDomains`: (optional, 기본 값 1) eligible domain의 최소 개수로 값은 0보다 커야하며 `whenUnsatisfiable`이 DoNotSchedule일 경우에만 사용할 수 있다. topology key와 매칭되는 eligible domain 개수가 `minDomains`보다 작으면 global minimum을 0으로 간주해 skew를 계산한다(global minimum은 eligible domain에서 매칭되는 po의 최소 개수). topology key와 매칭되는 eligible domain 개수가 `minDomains`와 같거나 더 크면 scheduling에 영향을 주지 않는다. domain의 개수가 `minDomains`보다 작으면 global minimum이 0이 되기 때문에 각 domain에는 최대 `maxSkew` 개수의 po만 스케줄링될 수 있다.
    - `maxSkew`: (required) po가 고르지 않게 분포될 수 있는 정도를 나타낸다. 이 필드는 필수이며 0 보다 큰 숫자를 사용해야 한다. 이 필드의 의미는 `whenUnsatisfiable` 필드 값에 따라 바뀐다.
      - `whenUnsatisfiable`이 DoNotSchedule: `maxSkew`는 대상 topology에 있는 매칭 po의 갯수와 global minimum(eligible domain 중 매칭되는 po 개수 값 중 가장 작은 수, 만약 eligible domain이 `minDomains`보다 작으면 0으로 간주) 사이의 최대 허용 차이를 정의한다. 예를 들어 2, 2, 1개의 매칭 po를 갖는 3개 zone이 있는 경우 `maxSkew`가 1이라면 global minimum은 1이다.
      - `whenUnsatisfiable`이 ScheduleAnyway: kube-scheduler는 skew를 줄이기 위해 도움이되는 topology에 더 높은 우선 순위를 부여한다.
    - `whenUnsatisfiable`: (required) spread constraint를 만족하지 않는 po를 처리할 방법을 설정한다.
      - DoNotSchedule: (default) 스케줄링을 수행하지 않는다.
      - ScheduleAnyway: skew를 최소화하는 no의 우선순위를 지정해 스케줄링을 계속 수행하도록 한다.
    - `labelSelector`: 매칭 po를 찾는데 사용된다. label selector에 매칭되는 po는 topology domain에 존재하는 po 개수를 계산하는데 사용된다.
    - `matchLabelKeys`: `labelSelector`와 마찬가지로 매칭 po를 찾는데 사용된다. 다만 label key의 유무만 검사한다. `labelSelector` 필드를 사용하는 경우에만 사용할 수 있으며 `labelSelector`와 중복되는 key를 사용할 수 없다. 그리고 `labelSelector`의 label 목록과 AND 연산 결과 label에 매칭되는 po를 찾는다. 빈 값은 모든 po를 의미하며 `labelSelector`만 고려한다. 예를 들어 deploy를 사용해 새로운 버전을 배포하는 경우에 pod-template-hash label을 사용해 revision을 고려할 수 있다.
    - `nodeAffinityPolicy`: topology spread skew를 계산할 때 po의 node affinity, node selector를 어떻게 처리할지 설정한다. 기본 값은 `Honor`다.
      - `Honor`: node affinity, node selector 결과 no에 대한 skew를 계산한다.
      - `Ignore`: 모든 no에 대한 skew를 계산한다
    - `nodeTaintsPolicy`: topology spread skew를 계산할 때 no의 taints를 어떻게 처리할지 설정한다. 기본 값은 `Ignore`다.
      - `Honor`: taints가 없는 no와 tainted no지만 po의 toleration이 있는 no에 대해 skew를 계산한다.
      - `Ignore`: 모든 no에 대한 skew를 계산한다.
- pod overhead는 보통 container runtime에서 container의 resource 외에 container를 생성 및 실행하는 데 필요한 리소스를 설정하기 위해 사용한다. pod overhead는 RuntimeClass admission controller가 po의 `.spec.overhead`을 추가한다. pod overhead는 스케줄링, eviction에서 모두 고려한다. ([Pod Overhead](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-overhead/))
- po는 생성되면 일단 스케줄링을 위한 준비가 완료된 것으로 간주된다. 그렇기 때문에 k8s scheduler는 즉시 pending pod를 배치할 no를 찾기 위해 노력한다. 이러한 po로 인해 원하지 않는 동작이 발생할 수 있다. 예를 들어 Cluster AutoScaler가 동작할 수도 있다. po의 `.spec.schedulingGate` 필드를 명시/삭제함으로써 po가 언제 scheduling 될 준비가 됐는지 제어할 수 있다. `.spec.schedulingGate` 필드에 문자열 목록을 명시할 수 있으며 각 문자열은 po가 스케줄링 될 준비가 됐다고 간주되기 전에 충족해야 하는 조건을 나타내는데 사용한다. 이 필드는 po가 생성될 때만 명시할 수 있다. 즉 po가 생성(schedulding이 아닌 kube-apiserver에 등록)된 후에는 각 필드를 임의의 순서로 제거할 수 있지만, 추가로 새로운 목록을 추가할 수는 없다. ([Pod Scheduling Readiness](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-scheduling-readiness/))
- node affinity는 po의 입장에서 `.spec.affinity.nodeAffinity` 필드를 사용해 해당 po가 스케쥴링될 no에 대한 제약 사항을 설정한다. 반면에 taint는 no의 입장에서 `.spec.taints` 필드를 사용해	 해당 no에 스케줄링, 실행될 수 있는 제약 사향을 설정한다. taint가 있는 no에는 `.spec.tolerations`에 해당 taint 목록이 있는 po만 스케쥴링, 실행될 수 있다. 그렇다고 toleration이 po의 무조건적인 스케줄링을 보장하는 것은 아니다. 왜냐하면 스케줄링을 위한 다른 평가 조건도 있기 때문이다. 만약 po에 `.spec.nodeName`를 명시하면 scheduler의 동작과 관련 없이 po는 해당 no에 binding된다(no에 `NoSchedule` taint가 있더라도). 물론 no에 `NoExecute` taint가 추가적으로 있는 경우에 kubelet이 po가 적절한 toleration이 없다면 실행하지 않는다. ([Taints and Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/))
  - po의 `.spec.tolerations[*]` 필드는 다음과 같다.
    - `tolerations[*].key`: 매칭시킬 taint의 key. 빈 값은 모든 taint를 의미하며 추가적으로 `tolerations[*].operator`가 `Exists`라면 모든 taint key와 value에 매칭된다.
    - `tolerations[*].operator`: 동일 key 내에서 value에 대한 추가 연산자. `Exists`와 `Equal`(기본 값)이 있다.
    - `tolerations[*].value`: 매칭시킬 taint의 value. `tolerations[*].operator`가 `Exists`라면 빈 값이어야 한다. 그렇지 않으면 정규 표현식으로 처리된다.
    - `tolerations[*].effect`: taint의 effect. `NoSchedule`, `PreferNoSchedule`, `NoExecute` 값을 사용할 수 있다. 빈 값은 모든 effect를 의미한다.
      - `NoExecute`: 새로운 po의 스케줄링도 제한하며 이미 실행 중인 po에도 영향을 미친다. 매칭되는 toleration이 없는 po는 즉시 eviction된다. 매칭되는 toleration에 대해 tolerationSeconds가 없는 po는 해당 no에 계속 bound된 상태로 남는다. 매칭되는 toleration에 대해 tolerationSeconds가 있는 po는 해당 시간 동안 bounde된 상태로 남아있으며 시간이 초과하면 eviction된다.
      - `NoSchedule`: 스케줄링 단계의 po에 영향을 미친다. 매칭되는 toleration이 없는 po는 해당 no에 스케줄링 될 수 없다. 이미 실행 중인 po는 eviction되지 않는다.
      - `PreferNoSchedule`: `NoSchedule`의 soft 버전이다. 시스템은 no의 taint를 허용하지 않는 po를 스케줄링하지 않으려고 노력하지만 반드시는 아니다.
    - `tolerations[*].tolerationSeconds`: `tolerations[*].effect`가 `NoExecute`일 때 eviction 대기 시간을 의미한다. 설정하지 않으면 eviction하지 않으며, 0은 즉시 eviction을 의미한다.
  - no의 `.spec.taints[*]` 필드 `effect`, `key`, `value`는 po의 toleration과 같은 의미다. 추가적으로 `timeAdded` 필드는 `NoExecute` effect에 대해 언제 추가됐는지를 나타내는 필드다.
- k8s는 사용자 또는 controller가 node.kubernetes.io/not-ready, node.kubernetes.io/unreachable toleration을 명시하지 않으면 tolerationSeconds=300와 같이 추가한다. 이는 no에 문제가 발생했을 때 5분 후에 해당 no에서 eviction되도록 하기 위한 조건이다. ([Taints and Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/#taint-based-evictions))
- ds는 po 생성 시 tolerationSeconds가 없는 NoExecute toleration `node.kubernetes.io/unreachable`, `node.kubernetes.io/not-ready`을 추가한다. 이를 통해 po가 eviction 되지 않도록 보장한다. ([Taints and Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/#taint-based-evictions))
- daemonset controller는 po 생성 시 아래 NoSchedule effct toleration을 추가한다. ([Taints and Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/#taint-nodes-by-condition))
  - node.kubernetes.io/memory-pressure
  - node.kubernetes.io/disk-pressure
  - node.kubernetes.io/pid-pressure (1.14 or later)
  - node.kubernetes.io/unschedulable (1.10 or later)
  - node.kubernetes.io/network-unavailable (host network only)
- po는 다른 po와의 상대적 중요성을 나타내기 위한 priority가 있다. po가 스케줄링될 수 없는 경우 kube-scheduler는 pending po보다 더 낮은 priority를 갖는 po를 preempt(evict)를 시도한다. k8s는 기본적으로 system-cluster-critical, system-node-critical pc를 제공한다. pc는 non-namespaced 리소스다. 우선 순위는 `.value` 필드에 정수 값을 설정한다. 값이 높을수록 우선 순위가 높다. 이름은 `system-`으로 시작하면 안된다. `.value` 필드는 1,000,000,000(1 billion) 이하의 32 bit 정수 값을 가질 수 있다. 즉 우선 순위 값은 -2147483648 <= 우선 순위 <= 1000000000 사이의 값을 갖는다. 더 큰 값은 system po를 위한 내장 pc를 위해 예약됐다. pc와 관련된 특이 사항은 다음과 같다. ([Pod Priority and Preemption](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/))
  - `.globalDefault` 필드는 `.spec.priorityClassName` 필드를 사용하지 않은 po에 사용될 기본 pc를 나타낸다. 만약 cluster에 기본 pc가 없는 경우 po들은 우선 순위 값 0을 갖는다.
  - cluster에 기본 pc를 생성한 경우 기존 po의 우선 순위는 변경되지 않으며 새로 생성되는 po에만 영향을 미친다.
  - 실행 중인 po들이 참조하는 pc를 삭제하는 경우, po가 참조하는 pc의 이름이 변경되지는 않지만 새로 생성되는 po는 삭제된 pc의 이름을 참조할 수 없다.
  - `.preemptionPolicy` 필드의 기본 값은 낮은 우선 순위를 갖는 po를 preempt할 수 있는 `PreemptLowerPriority`이다. `Never` 값의 pc를 갖는 po는 우선 순위가 낮은 po보다 scheduling queue에서 더 앞에 위치하게 되지만 다른 po를 preempt할 수 없다. 이러한 non-preempting po는 충분한 resource가 확보될 때까지 scheduling queue에 대기한다. non-preempting po는 다른 po와 마찬가지로 scheduler의 back-off의 대상이 된다.
  - non-preempting po는 다른 po를 preempt할 수 없지만 우선 순위가 높은 po에 의해 자신은 preempt될 수 있다.
  - po를 생성할 때 `.spec.priorityClassName` 필드를 통해 pc를 참조함으로써 po의 우선 순위를 명시할 수 있다. priority admission controller는 이 필드를 사용해 실제 우선 순위를 나타내는 정수 값을 `.spec.priority` 필드에 할당한다. 만약 po가 유효하지 않은 pc를 참조하는 경우 po는 reject된다.
- containerfs(container 파일시스템)에 대한 지원을 활성화하는 split image filesystem 기능은 새로운 eviction signals, thresholds, metrics을 추가한다. containerfs를 사용하려면 Kubernetes v1.32에서 `KubeletSeparateDiskGC` feature gate를 활성화해야 한다. 현재 containerfs 지원은 CRI-O(v1.29 이상)에서만 제공된다.
---

- no의 graceful/non-graceful shutdown 설정 고려 ([Node Shutdowns](https://kubernetes.io/docs/concepts/cluster-administration/node-shutdown/))
- kubelet이 특정 label을 마음대로 수정할 수 없도록 NodeRestriction admission plugin 설정 고려 ([Assigning Pods to Nodes](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#node-isolation-restriction))
- kubelet이 image credential provider을 사용해 동적으로 credential을 얻도록 사용 고려([Extending Kubernetes](https://kubernetes.io/docs/concepts/extend-kubernetes/#kubelet-image-credential-provider-plugins))
- kubelet의 authentication, authorization 설정 고려([Kubelet authentication/authorization](https://kubernetes.io/docs/reference/access-authn-authz/kubelet-authn-authz/))
- kubelet의 client certificate rotation은 `.rotateCertificates` 필드, server certificate 요청 및 rotation은 `.serverTLSBootstrap` 필드 사용([TLS bootstrapping](https://kubernetes.io/docs/reference/access-authn-authz/kubelet-tls-bootstrapping/#certificate-rotation))

### addon
- 구성 요소: `dns`, `web ui`, `container resource monitoring`, `cluster-level monitoring`
- addon의 기본 limit은 일반적으로 작은 크기의 cluster에서의 경험을 통해 수집한 데이터를 기반으로 하기 때문에 조정 필요 ([Considerations for large clusters](https://kubernetes.io/docs/setup/best-practices/cluster-large/#addon-resources))
- cluster의 사이즈가 커짐에 po의 resource 증량이 필요할 수 있으며, vpa의 recommender mode를 이용해 리소스의 제안 용량을 확인한다. ([Considerations for large clusters](https://kubernetes.io/docs/setup/best-practices/cluster-large/#addon-resources))
---

- k8s object의 status를 노출하는 kube-state-metrics 설치 고려 ([Metrics for Kubernetes Object States](https://kubernetes.io/docs/concepts/cluster-administration/kube-state-metrics/))

## EKS
- control plane은 kube-apiserver, etcd 구성 요소들로 이뤄지며 aws가 관리하는 ec2 인스턴스에서 실행된다. control plane은 multi az에 provision되며 nlb를 통해 kube-apiserver를 노출한다. ([Clusters](https://docs.aws.amazon.com/eks/latest/userguide/clusters.html))
  - 확인 시 eks 생성 시 설정하는 subnet 개수가 3, 4개여도 항상 두 개의 az를 사용하는 nlb를 생성하기 때문에 2개의 ip를 통해 kube-apiserver에 접근할 수 있다.
- eks는 2개의 vpc로 구성된다. ([VPC and Subnet Considerations](https://docs.aws.amazon.com/eks/latest/best-practices/subnets.html#_overview))
  - aws-managed vpc: control plane 구성요소가 실행된다. 사용자 aws 계정에 보이지 않음
  - customer-managed vpc: k8s no가 실행된다.
- etcd node의 모든 저장 데이터는 aws ebs volume를 통해 저장되며 kms로 암호화된다. ([Clusters](https://docs.aws.amazon.com/eks/latest/userguide/clusters.html))
- eks는 etcd storage 크기를 8GiB로 설정한다. 이는 일반적인 환경에서 etcd의 최대 권장 사이즈다. ([Clusters](https://docs.aws.amazon.com/eks/latest/userguide/clusters.html))
- eks cluster insight는 eks, k8s의 best practice를 따를 수 있도록 권장 사항을 제공한다. 이를 위해 eks cluster에 대해 반복적으로 검사를 수행한다. eks를 업데이트 하기 전에 cluster insight를 확인하는 것을 권장한다. ([Cluster insights](https://docs.aws.amazon.com/eks/latest/userguide/cluster-insights.html))
- cluster의 public/private endpoint access 조합에 따라 cluster가 생성된 vpc와 인터넷에서 kube-apiserver에 접근하는 방법이 다르다. 두 옵션을 모두 비활성화 할 수는 없다. kube-apiserver가 no에 대한 접근(예를 들어 사용자의 `kubectl exec`, `kubectl logs`, `kubectl proxy` 명령어)을 위해서 기본적으로 cluster 생성시 명시한 subnet에 eks-managed eni을 생성하며 이는 public/private endpoint access 옵션과 관련 없다. ([AWS Blogs](https://aws.amazon.com/ko/blogs/containers/de-mystifying-cluster-networking-for-amazon-eks-worker-nodes/))
- eks의 control plane 구성 요소는 aws가 관리하는 vpc에서 생성되며 kube-apiserver의 엔드포인트 역할을 수행하는 eks-managed eni는 eks 생성 시 설정한 subnet에 랜덤하게 생성된다. ([AWS blog](https://aws.amazon.com/ko/blogs/containers/de-mystifying-cluster-networking-for-amazon-eks-worker-nodes/))
- eks private endpoint는 aws privatelink endpoint가 아니기 때문에 privatelink endpoint 목록에서 조회 불가능하다. private access endpoint를 활성화하면 aws eks는 route 53 private hosted zone을 사용자 대신 생성하고 cluster가 생성된 vpc와 연결한다. 이 private hosted zone은 사용자가 조회할 수 없으며 eks가 대신 관리한다. vpc에서 private hosted zone을 통해 kube-apiserver에 접근하기 위해 사용자 vpc는 `enableDnsHostnames`, `enableDnsSupport`을 true로 설정하고 DHCP optionset은 dns server 목록에 `AmazonProvidedDNS`을 포함해야 한다. ([Configure endpoint access](https://docs.aws.amazon.com/eks/latest/userguide/cluster-endpoint.html#cluster-endpoint-private))
- public/private endpoint 활성화/비활성화 여부에 따른 특징은 다음과 같다. ([Configure endpoint access](https://docs.aws.amazon.com/eks/latest/userguide/cluster-endpoint.html#modify-endpoint-access))
  - enabled/disabled
    - eks의 기본 설정 값이다. vpc 내에서 kube-apiserver에 대한 요청은 vpc를 벗어나지만 aws 네트워크를 벗어나지는 않는다.
    - 인터넷에서 public endpoint를 통해 kube-apiserver에 접근할 수 있다. public endpoint에 대해 CIDR block을 이용한 접근 제어가 가능하다.
  - enabled/enabled
    - vpc 내에서 kube-apiserver에 대한 요청은 vpc 내 eks가 생성한 eks-managed eni를 향한다.
    - 인터넷에서 public endpoint를 통해 kube-apiserver에 접근할 수 있다. public endpoint에 대해 CIDR block을 이용한 접근 제어가 가능하다.
  - disabled/enabled
    - kube-apiserver에 대한 모든 요청은 vpc 내부 또는 connected network 내부에서만 가능하다.
    - 인터넷에서 kube-apiserver에 접근할 수 없다.
    - cluster의 kube-apiserver endpoint는 public DNS에 의해 vpc 내 private ip 주소로 resolving된다.
- outbound internet access가 필요없는 eks(private cluster)를 배포하기 위해 아래 조건을 만족해야 한다. ([Private clusters](https://docs.aws.amazon.com/eks/latest/userguide/private-clusters.html))
  - cluster는 container image를 vpc 내 registry에서 pull할 수 있어야 한다.
  - eks는 private endpoint가 활성화 돼야한다. public endpoint 활성화는 optional이다.
  - self-managed node의 경우 kubelet의 `--apiserver-endpoint`, `--b64-cluster-ca` flag를 사용해 aws eks API에 대한 접근, eks introspection 과정을 생략한다.
  - cluster의 `aws-auth` cm은 vpc 내에서 생성돼야 한다.
  - irsa를 사용하는 경우 po는 aws sts API 호출을 통해 credential을 획득한다. 접근할 수 있도록 sts에 대한 vpc endpoint를 생성해야 한다. 대부분의 v1 sdk는 global aws sts endpoint를 기본 값으로 `sts.amazonaws.com`로 사용하며 aws sts vpc endpoint를 사용하지 않는다. aws sts vpc endpoint 사용을 위해 sdk의 설정을 변경해야 한다.
  - po가 aws service에 접근할 수 있도록 vpc endpoint가 있어야 한다.
  - self-managed node는 vpc endpoint가 존재하는 subnet에 배포돼야 한다. managed node group의 경우 vpc endpoint의 security group inbound rule에 managed node group이 배포된 subnet을 추가해야 한다.
  - efs volume을 사용하는 경우 aws-efs-csi-driver의 container image가 eks cluster와 동일한 region을 사용하도록 변경해야 한다.
  - aws-load-balancer-controller를 배포할 때 `enable-shield`, `enable-waf`, `enable-wafv2` flag를 false로 설정해야 한다. 그리고 certificate discovery 기능도 사용할 수 없다. 왜냐하면 aws certificate manager가 vpc endpoint를 지원하지 않기 때문이다.
  - cluster autoscaler po를 배포할 때 `--aws-use-static-instance-list=true` flag를 포함해야 한다. 그리고 vpc는 sts vpc endpoint, autoscaling vpc endpoint를 포함해야 한다.
  - 일부 container software의 경우 사용량 모니터링을 위해 aws marketplace metering server 접근을 위한 API 호출을 수행한다. private cluster에서 해당 호출을 허용하지 않기 때문에 이러한 유형의 container는 private cluster에서 사용이 불가하다.
- eks platform version은 eks control plane의 기능을 나타내며 kube-apiserver의 flag 활성화, k8s patch 버전 등을 포함한다. k8s minor version마다 1개 이상의 eks platform version을 갖는다. k8s minor version 별로 eks platform version은 독립적이다. k8s 1.32 version의 첫 eks platform version은 `eks.1`이다. eks는 주기적으로 새로운 platform version을 release해 control plane의 새로운 설정 활성화, 보안 수정을 제공한다. eks는 기존 cluster의 platform version을 자동으로 업그레이드 한다. cluster가 최신 platform version 보다 두 개 이상 차이나면 자동으로 업데이트 하지 못할 수도 있다. eks는 새로운 ami를 같이 release할 수 있지만 동일한 k8s minor version 내에서는 eks control plane과 no의 ami 간 호환성을 보장한다. 새로운 platform version은 기존 서비스에 영향을 주지 변경 사항을 도입하지 않는다. 새로운 cluster 생성 시 해당 k8s minor version 내에서 가장 최신의 eks platform 버전으로 자동 생성된다. ([Platform versions](https://docs.aws.amazon.com/eks/latest/userguide/platform-versions.html))
- Managed Access 정리
- node group은 k8s의 native resource는 아니며 여러 특성을 공유하는 no의 집합을 의미하는 추상적인 개념이다. eks는 managed node group과 self-managed node 개념을 사용한다. managed node group은 auto scaling group을 사용하며 no의 관리 편의성을 위한 추가적인 기능(no 버전 업그레이드, graceful no termination)을 제공한다. self-managed node는 managed node group을 사용하지 않은 형태로 사용자가 구현하는 방식에 따라 다르다. 예를 들어 auto scaliing group을 통해 no로 사용될 ec2 집합을 관리하거나 ec2 인스턴스를 직접 관리할 수도 있다. self-managed node의 경우 eks를 위한 필수 tag를 추가해야 한다. `eksctl`을 통해 두 유형의 node group을 쉽게 관리할 수 있다. ([Cluster Autoscaler](https://docs.aws.amazon.com/eks/latest/best-practices/cas.html#_overview))
- aws는 k8s cluster autoscaler가 auto scaling group의 `.DesiredReplicas`를 제어해 scaling을 수행하도록 구현했다. karpenter가 존재하기 전 사용자는 no의 scaling을 제어하기 위해 auto scaling group이나 k8s cluster autoscaler에 의존해야 했다. karpenter는 auto scaling group과 managed node group을 사용하지 않으며 k8s native API와 더 밀접하게 스케일링 관리를 수행한다. auto scaling group과 managed node group는 aws-native 추상화 계층 역할을 수행해 ec2 cpu 부하와 같은 aws 수준의 metric을 기반으로 스케일링을 수행한다. cluster autoscaler는 k8s의 추상화를 aws 추상화로 변환하는 역할을 하지만 특정 az에 대한 스케줄링과 같은 유연성을 제공하지 않는다. karpenter는 aws의 한 계층을 제거해 k8s 내에서 직접 유연성을 제공할 수 있도록 한다. 이는 급격한 수요 변화가 발생하거나 다양한 컴퓨팅 요구 사항이 있는 워크로드를 실행하는 cluster에서 가장 효과적으로 사용할 수 있다. 반면, auto scaling group과 managed node group은 보다 정적이고 일관된 워크로드를 실행하는 cluster에 적합하다. ([Karpenter](https://docs.aws.amazon.com/eks/latest/best-practices/karpenter.html#_karpenter_best_practices))
  - karpenter는 k8s의 표준 pod scheduling constraints(resource request, node selector, node affinity, pod affinity/anti-affinity, topology spread)을 지원하며 nodepool의 제약 조건에 매칭되어야 node가 provision될 수 있다. 아래는 karpenter가 no를 provision 하기 위해 pod scheduling constraints을 다루는 방법이다. ([Scheduling](https://karpenter.sh/docs/concepts/scheduling/))
    - 기본적으로 karpenter는 preference(node affinity, pod affinity/anti-affinity의 `preferredDuringSchedulingIgnoredDuringExecution`, pod topology의 `ScheduleAnyway`)를 요구 사항(각각 `requiredDuringSchedulingIgnoredDuringExecution`, `DoNotSchedule`)으로 간주해 pod scheduling을 생각한다.
    - karpenter는 preferred affinities(node affinity, pod affinity/anti-affinity의 `preferredDuringSchedulingIgnoredDuringExecution`)을 required affinities로 간주하지만 scheduling이 불가능한 경우 weight가 낮은 조건부터 1개씩 완화하면서 다시 scheduling 가능한지 고려한다.
    - karpenter는 preferred affinities가 pod topology와 같이 있을 경우 required affinities로 간주하지 않는다.
    - disruption controller는 automated method(drift -> consolidation 순서)를 실행해 disruptable node를 찾는다. consolidation은 node를 삭제(deletion)/교체(replace)하는 작업을 수행한다. 3가지 메커니즘을 순서대로 실행해 작업을 수행한다.
      1. empty node consolidation: 빈 node(ds가 아닌 po가 존재하지 않는 경우)를 삭제
      2. multi node consolidation: 여러 node를 가격이 더 낮은 1개의 node로 교체. 여러 node에 대한 consolidation은 모든 가능한 조합을 검토하는 것은 비현실적이기 때문에 heuristic(경험)을 사용해 consolidation이 가능성이 높은 여러 node를 식별한다. 여러 node가 삭제 또는 교체될 수 있는 경우 사용자의 workload를 최소화하는 node를 우선적으로 선택한다.
        - 실행 중인 po가 적은 node
        - 곧 만료 예정인 node
        - 우선 순위가 낮은 po를 실행 중인 node
      3. single node consolidation: 1개의 node를 가격이 더 낮은 1개의 node로 교체
    - consolidation은 node를 통합하는 작업이다. karpenter는 1개의 node를 총합의 가격이 낮은 여러개의 node로 분리하는 작업은 수행하지 않는다. ([GitHub Issuse](https://github.com/aws/karpenter-provider-aws/issues/5304))
- hybrid node를 제외하고 node는 cluster 생성 시 명시한 subnet과 동일한 vpc에 존재해야 한다(동일한 subnet에 존재할 필요는 없다). ([Manage compute](https://docs.aws.amazon.com/eks/latest/userguide/eks-compute.html))
- eks는 eks optimized ami를 제공한다. 이 ami은 eks 관련 `containerd`, `kubelet`, `aws iam authenticator` 구성요소를 포함하며 eks control plane을 찾아 연결을 시도하는 특별한 `bootstrap.sh`, `max-pods-calculator.sh` script(al2 기준)를 포함한다. script의 argument를 이용해 kubelet의 flag를 설정할 수 있다. ([Self-managed nodes](https://docs.aws.amazon.com/eks/latest/userguide/worker.html))
- managed node group의 특성은 다음과 같다. ([Managed node groups](https://docs.aws.amazon.com/eks/latest/userguide/managed-node-groups.html))
  - 사용자 대신 no의 provision과 lifecycle 관리를 수행한다. 뿐만 아니라 ec2의 업데이트와 종료 시 사용자 대신 자동으로 no를 drain한다.
  - auto scaling group을 이용해 no를 사용자가 정의한 az에 고루 분포되도록 실행한다(az rebalance 기능).
  - no의 health 모니터링 및 auto repair와 같은 기능을 제공한다.
  - eks는 managed node group의 no가 cluster autoscaler에 의해 auto-discovery 대상이 될 수 있도록 자동으로 tag를 추가한다.
  - managed node 생성 시 custom launch template을 사용할 수 있다. 사용하지 않을 경우 eks가 기본 launch template을 생성 및 사용한다. 자동 생성된 launch template을 수정하는 것, 나중에 node group 생성시 사용하는 것을 권장하지 않으며 오류가 발생할 수 있다.
  - eks optimized ami를 사용하는 경우 새로운 릴리즈가 있을 경우 자동으로 업데이트를 수행하지는 않으며 사용자가 변경된 ami를 사용하도록 managed node group 업데이트를 수행해야 한다.
  - aws outpost, wavelength에 배포할 수 없으며(self-managed node는 가능) local zone에는 가능하다.
  - ec2 instsance status check에 실패하는 경우 eks는 에러 코드를 반환한다.
  - eks는 `eks.amazonaws.com` label을 no에 추가한다.
  - managed node group의 업데이트는 po의 pdb를 존중하지만 auto scaling group의 az rebalance 작업, 사용자가 desired capacity를 줄이는 작업은 pdb를 존중하지 않을 수 있다. 이러한 작업은 auto scaling group의 scale-down call을 통해 즉시 node를 종료되게끔 한다. 이로인해 pdb와 관계 없이 pod는 즉시 eviction 시도되며 만약 pod eviction에 15분이 경과한 경우 pod의 종료 여부와 관계 없이 node가 종료된다. node의 종료 시간을 늘리기 위해 lifecycle hook을 추가할 수 있다.
    - auto scaling group은 rebalance와 관련해 az rebalance(기본 기능), capacity rebalance(활성화 필요) 기능을 제공한다. az rebalance는 각 az에 대한 ec2 배포가 고르지 않을 경우 발생한다. 기본적으로 기존 ec2를 종료하기 전에 새로운 ec2를 띄우는 작업을 수행한다.
  - spot interruption notification, capacity rebalance notification(rebalance recommendation)을 수신한 후 no drain 과정이 정상적으로 수행되기 위해 auto scaling group의 capacity rebalance 기능을 활성화 할 수 있다.
    - capacity rebalance 기능은 instance purchage option에서 spot instance의 비율(managed node group의 경우 on-demand와 spot을 같이 지정할 수 없다)을 지정할 경우에만 사용할 수 있다. aws ec2 서비스는 인스턴스를 interruption 하기 전에 항상 spot two-minute instance interruption notice과 rebalance recommendation(중단 위험이 높은 경우)를 제공한다(ec2 서비스는 항상 spot interruption notification보다 rebalance recommendation을 먼저 보낸다는 보장은 없으며 동시에 도착할 수도 있다). capacity rebalancing은 rebalance recommendation을 받은 spot 인스턴스를 사전에 교체하려고 시도한다. 이를 통해 새로운 spot 인스턴스로 부하를 분산할 수 있는 기회를 제공한다. 새로운 spot 인스턴스의 health check를 통과한 후에 기존 spot 인스턴스를 종료하기 시작한다. capacity rebalancing을 사용하지 않으면 auto scaling group은 ec2 spot 서비스가 인스턴스를 중단시키고 health check에 실패한 후에야 spot 인스턴스를 교체한다.
    - 교체 spot node가 Ready 상태가 되면 eks는 기존 node를 cordon, drain하는 정상적인 과정을 거치지만 새롭게 교체할 spot no가 Ready 상태가 되기전에 spot two-minute instance interruption notice를 수신하면 rebalance recommendation을 수신한 spot 인스턴스를 즉시 drain한다. eks는 최선을 다하지만 기존 no를 drain하기 전에 교체 node가 cluster에 조인할 때까지 기다린다는 보장은 없다.
  - launch template을 이용해 no의 ebs를 암호화할 수 있다.
  - capacity type을 on-demand로 지정(기본 값)하는 경우 managed node group은 사용자 대신 아래와 같은 설정을 적용한다.
    - prioritized allocation strategy를 사용한다. 사용자가 인스턴스 타입 목록으로 c5.large, c4.large, c3.large으로 지정하는 경우 먼저 명시된 타입부터 사용한다.
    - 모든 no에 `eks.amazonaws.com/capacityType: ON_DEMAND` label을 추가한다.
  - capacity type을 spot으로 지정하는 경우 managed node group은 사용자 대신 아래와 같은 설정을 적용한다.
    - spot capacity pool(동일 az와 인스턴스 타입을 갖는 사용되지 않는 ec2 인스턴스 집합)에서 최적의 spot node를 provision하기 위해 k8s 1.28 이후 버전부터는 allocation strategy을 pco(`price-capacity-optimized`)로 설정한다.
    - 모든 no에 `eks.amazonaws.com/capacityType: SPOT` label을 추가한다.
  - spot 인스턴스를 사용하는 경우 가용성을 최대로 보장하기 위해 여러 인스턴스 타입 사용을 권장한다.
  - custom launch template을 사용해 사용자는 node group에 대한 상세 커스터마이징을 수행할 수 있다.
  - custom launch template에 ami를 지정하지 않고 managed node group을 만드는 경우(즉 managed node group 생성 시 설정한 eks optimized ami를 사용) eks optimized ami에 내장된 `bootstrap.sh`(al2 기준)은 인스턴스 타입에 따른 `.maxPods`(kubelet flag) 값을 계산하고 vCPU 개수에 따라 최대 값을 제한한다(30개 이하인 경우 110, 30개보다 많은 경우 250으로 제한). managed node group의 ami 타입 설정을 사용하지 않고 launch template에 ami id를 명시하는 경우 사용자는 `bootstrap.sh`(al2) 또는 kubelet 명령어를 사용해 cluster에 join할 수 있도록 해야 한다. ([Create](https://docs.aws.amazon.com/eks/latest/userguide/create-managed-node-group.html#eksctl_create_managed_nodegroup))
  - custom launch tempalte을 사용하는 경우 아래 조건을 만족하는 경우 imds를 비활성화하는 것을 권장한다(확인 시, eks api가 생성하는 launch template에는 imds가 비활성화 되어 있다).
    - sa에 iam role을 할당하고 po가 최소한의 권한만 갖도록 설정이 필요한 경우
    - po가 ec2 imds 접근이 필요하지 않는 경우(예를 들어 region 확인)
  - custom launch template에 ami를 명시하지 않는 경우, managed node group은 인스턴스 타입에 따른 최대 po 개수를 계산한다(kubelet의 `.maxPods` 값 계산 및 사용). al2의 경우 launch template 내 `bootstrap.sh`에서 수행하지만 al2023의 경우 launch template에 계산된 값이 설정된다(만약 여러 인스턴스 타입을 사용하는 경우 가장 작은 값이 모든 node에 적용). ([Create](https://docs.aws.amazon.com/eks/latest/userguide/create-managed-node-group.html#eksctl_create_managed_nodegroup))
    - al2023의 경우 eks가 생성하는 launch template 내 user data는 다음과 같다. 즉 managed node group에 사용될 여러 인스턴스 타입의 maxPods 값 중 가장 작은 값을 user data에 사용한다.
      ``` yaml
      apiVersion: node.eks.aws/v1alpha1
      kind: NodeConfig
      spec:
        cluster:
          name: my-cluster
          apiServerEndpoint: https://example.com
          certificateAuthority: Y2VydGlmaWNhdGVBdXRob3JpdHk=
          cidr: 10.100.0.0/16
        kubelet:
          config:
            maxPods: 35
            clusterDNS:
              - 10.100.0.10
          flags:
            - "--node-labels=eks.amazonaws.com/sourceLaunchTemplateVersion=1,eks.amazonaws.com/nodegroup-image=ami-00aa0ede823b748b9,eks.amazonaws.com/capacityType=ON_DEMAND,eks.amazonaws.com/nodegroup=my-cluster-ng,eks.amazonaws.com/sourceLaunchTemplateId=lt-0df8d400bb6b40a81"
      ```
  - custom launch template에 ami를 명시하는 경우, 최대 po 개수를 명시해야 한다. ([Create](https://docs.aws.amazon.com/eks/latest/userguide/create-managed-node-group.html#eksctl_create_managed_nodegroup))
  - 다음 상황에서 managed node group 업데이트를 수행할 수 있다: cluster 버전 업데이트에 따른 no 버전 업데이트 / 새로운 ami 버전 적용 / managed node group의 minimum, maximum, desired count 조정, managed node group의 no에 k8s label 수정 / managed node group의 aws tag 수정 / launch template 변경 적용. launch template 없이 생성한 managed node group을 새로운 launch template 버전을 사용하도록 직접 업그레이드를 할 수 없으며 대신 새로운 launch template을 사용하는 managed node group을 새로 생성해야 한다. ([Update](https://docs.aws.amazon.com/eks/latest/userguide/update-managed-node-group.html))
  - managed node group의 업그레이드는 4개의 phase를 수행한다. managed node group의 모든 변경이 아래 단계를 수행하는 것은 아니다. 예를 들어 label 수정과 같은 작업은 launch template 버전을 새롭게 생성하지만 새로운 node를 띄우는 작업부터는 수행하지 않으며 기존 node에 수정된 label 정보를 반영한다. ([Update behavior details](https://docs.aws.amazon.com/eks/latest/userguide/managed-node-update-behavior.html))
    - setup phase
      - auto scaling group의 launch template 버전을 새로 생성한다.
      - auto scaling group이 새로운 launch template 버전을 사용하도록 수정한다.
      - managed node group의 updage 설정을 고려해 병렬로 업그레이드 수행할 최대 no의 개수를 결정한다.
    - scale up phase: 새롭게 생성되는 no를 기존 no와 동일한 az에 배치하기 위해 auto scaling g~r~oup의 az reblanace 기능을 사용한다. managed node group의 update strategy가 default일 경우 scale up phase을 수행하지만 minimal일 경우에는 scale up phase는 수행되지 않는다.
      - auto scaling group의 maximum size, desired size에 max(auto scaling group의 az 수 * 2, maxUnavailable) 값을 더한 값으로 변경한다. 계산식 중 az 수 * 2의 의미는 az rebalance를 보장하기 위해 각 az에 최소 2개의 추가 no 생성한다.
      - maximum size, desired size 값을 변경한 후 새로운 no가 존재하는지 확인한다. 이 단계는 아래 조건들을 만족하는 경우 성공한다.
        - 각 az에 적어도 1개의 새로운 no가 실행됐다.
        - 새로운 no가 모두 `Ready` 상태이다.
        - 새로운 no가 아래 label을 갖는다.
          - 일반적인 node group의 경우 eks는 아래 label을 추가한다.
            - `eks.amazonaws.com/nodegroup-image=$amiName`
            - `eks.amazonaws.com/nodegroup=$nodeGroupName`
          - custom launch template이나 custom ami를 사용하는 node group의 경우 eks는 아래 label을 추가한다.
            - `eks.amazonaws.com/nodegroup-image=$amiName`
            - `eks.amazonaws.com/nodegroup=$nodeGroupName`
            - `eks.amazonaws.com/sourceLaunchTemplateId=$launchTemplateId`
            - `eks.amazonaws.com/sourceLaunchTemplateVersion=$launchTemplateVersion`
      - 기존 no에 po가 스케줄링되지 않도록 taint를 추가하고 no를 종료하기 전에 load balancer에서 제거하기 위해 `node.kubernetes.io/exclude-from-external-load-balancers=true` label을 추가한다.
    - upgrade phase: managed node group의 update strategy에 따라 다르게 동작한다. default strategy는 기존 no를 종료하기 전에 새로운 no를 먼저 생성(scale up phase)해 가용성을 보장한다는 장점이 있다. minimal strategy의 경우에는 no를 먼저 생성하지 않으며 새로운 no를 생성하기 전에 기존 no를 종료한다.
      - default update strategy
	      1. auto scaling group의 desired count를 증가하여 새 no를 생성한다(scale up phase).
	      2. 업그레이드가 필요한(죵료) no를 랜덤하게 선택한다. (최대 maxUnavailable 값까지)
	      3. 해당 no를 drain해 po를 종료한다. po가 15분 내에 모두 종료되지 않으면(예를 들어 pdb로 인해) `PodEvictionFailure` 오류 발생한다. 이를 방지하기 위해 managed node upgrade 요청 시 강제(force) 옵션을 사용할 수 있다.
	      4. 모든 po가 종료된 후 no를 cordon(예약 불가 상태)으로 설정하고 60초 대기한다. This is done so that the service controller doesn’t send any new requests to this node and removes this node from its list of active nodes.
	      5. auto scaling group에 no 종료 요청을 전송한다.
	      6. 위 단계를 반복해 이전 버전의 no가 모두 교체될 때까지 진행된다.
      - minimal update strategy
	      1. 업그레이드할 no를 랜덤하게 선택한다. (최대 maxUnavailable 값까지)
	      2. 해당 no를 drain해 po를 종료한다. po가 15분 내에 모두 종료되지 않으면(예를 들어 pdb로 인해) `PodEvictionFailure` 오류 발생한다. 이를 방지하기 위해 managed node upgrade 요청 시 강제(force) 옵션을 사용할 수 있다.
	      3. 모든 po가 종료된 후 no를 cordon(예약 불가 상태)으로 설정하고 60초 대기한다. This is done so that the service controller doesn’t send any new requests to this node and removes this node from its list of active nodes.
	      4. auto scaling group에 no 종료 요청을 전송한다. 그리고 auto scaling group은 새로운 no를 생성한다.
	      5. 위 단계를 반복해 이전 버전의 no가 모두 교체될 때까지 진행된다.
    - scale down phase
      - 업데이트 전의 상태로 되돌리기 위해 auto scaling group의 maximum size, desired size를 원래 값으로 변경한다. If the Upgrade workflow determines that the Cluster Autoscaler is scaling up the node group during the scale down phase of the workflow, it exits immediately without bringing the node group back to its original size.
  - managed node group을 삭제하면 eks는 가장 먼저 auto scaling group의 minimum, maximum, desired size를 0으로 설정한다. 각 인스턴스가 terminate 되기 전에 eks는 drain을 수행한다. po가 몇 분이 지나도 drain되지 않으면 eks는 auto scaling group을 통해 인스턴스를 종료한다. 모든 인스턴스가 terminate되고 auto scaling group이 삭제된다. ([Delete](https://docs.aws.amazon.com/eks/latest/userguide/delete-managed-node-group.html))
  - custom launch template을 사용해 managed node group을 배포할 수 있다. 이를 통해 사용자는 kubelet의 flag(eks optimized ami) 설정, no가 속한 subnet과 다른 cidr를 통한 po ip 할당, 사용자 정의 ami 설정, 사용자 정의 cni 배포를 수행할 수 있다. 뿐만 아니라 다른 버전의 launch template로 변경해 managed node group을 업데이트할 수 있다. custom launch template을 명시하지 않을 경우 eks는 기본 값을 갖는 launch template을 사용하며 사용자가 직접 수정하지 않는 것을 권장한다. custom launch template을 사용하는 managed node group은 직접 업데이트할 수 없다. 대신 custom launch template을 사용하는 신규 managed node group을 만들어야 한다. ([Launch templates](https://docs.aws.amazon.com/eks/latest/userguide/launch-templates.html))
  - custom launch template을 사용하는 경우 일부 설정이 managed node group 설정과 대응되는 부분이 있다. 그렇기 때문에 사용자는 launch template 또는 managed node group에 설정해야 하는 정보를 알아야 하며 두 곳에 모두 설정 가능한 경우 한 곳에만 설정해야 한다. 이를 어길 경우 managed node group의 생성, 업데이트가 실패하게 된다. ([Launch templates](https://docs.aws.amazon.com/eks/latest/userguide/launch-templates.html#launch-template-basics))
  - custom launch template에 인스턴스 타입을 0-1개만 지정할 수 있다. managed node group 설정에서는 0-20개의 인스턴스 타입을 지정할 수 있다. 만약 두 곳에서 모두 지정하지 않으면 기본적으로 t3.medium을 사용하게 된다. ([Launch templates](https://docs.aws.amazon.com/eks/latest/userguide/launch-templates.html#launch-template-basics))
  - launch template은 인스턴스 레벨의 security group 또는 eni 레벨의 security group을 지정할 수 있다. 만약 launch template에 security group을 설정할 경우 eks는 managed node group의 node에 cluster security group을 추가하지 않기 때문에 eks control plane과의 정상적인 통신을 위해 security group에 관련 규칙도 추가해야 한다. ([Launch templates](https://docs.aws.amazon.com/eks/latest/userguide/launch-templates.html#launch-template-security-groups))
  - launch template 내 user data는 `cloud-init`을 사용한다. managed node group에 사용되는 launch template의 user data는 amazon linux ami의 경우 MIME multi-part archive, Bottlerocket ami의 경우 TOML 포맷이어야 한다. 왜냐하면 eks는 no가 cluster에 join하는 데 필요한 eks user data(eks가 생성)와 병합하기 때문이다. 추가적으로 kubelet을 시작하거나 수정하는 명령어를 사용하면 안된다. no가 cluster에 join하기 위한 kubelet 관련 작업은 eks user data에서 자동으로 수행하기 때문이다. launch template에 ami를 설정하는 경우에는 eks user data를 병합하지 않기 때문에 no가 cluster에 join할 수 있도록 user data에 kubelet 실행 관련 명령어를 실행해야 한다(이 경우 ami에 맞는 형식을 따를 필요가 없음). 이 경우 사용자는 kubelet 명령어에 여러 flag를 설정할 수 있다. ([Launch templates](https://docs.aws.amazon.com/eks/latest/userguide/launch-templates.html#launch-template-user-data))
  - eks가 생성한(필요할 경우 병합) user data는 콘솔에서 확인이 가능하다.
  - launch template에 ami를 지정하는 경우 아래와 같은 제한이 있다. ([Launch templates](https://docs.aws.amazon.com/eks/latest/userguide/launch-templates.html#mng-ami-id-conditions))
    - launch template 내 ami를 지정과 지정하지 않는 것에 대한 변경은 불가하며 이를 위해 신규 node group을 만들어야 한다.
    - aws console에서 새로운 ami 버전에 대한 알림을 받을 수 없다.
    - managed node group의 taint 설정이 비동기적으로 설정될 수 있다. no가 cluster에 join하기 전에 taint를 적용하기 위해서는 kubelet 명령어에 `--register-with-taints` flag를 사용해야 한다.
    - imds가 2로 설정된다.(활성화/비활성화는 확인 필요)
- 여러 ec2 인스턴스 타입에 대한 지원을 위해 eks optimized ami의 여러 release를 제공한다. eks에서 지원하는 인스턴스 타입을 선택하기 위해 아래 내용을 고려한다. ([Amazon EC2 instance types](https://docs.aws.amazon.com/eks/latest/userguide/choosing-instance-type.html))
  - Number of instances in a node group: 일반적으로 ds가 많은 경우 큰 인스턴스를 적게 운영하는 것이 좋다.
  - Operating system: Linux, Windows, Bottlerock에 따라 지원하는 인스턴스 타입을 확인 및 리뷰한다.
  - Hardware architecture: eks optimized ami에서 지원하는지 확인한다.
  - Maximum number of Pods: vpc cni를 사용하는 경우 각 po는 ip를 할당받는다. 인스턴스 타입마다 제공하는 ip의 개수가 인스턴스에서 실행할 수 있는 최대 po의 개수를 결정하는 요소다. `max-pods-calculator.sh`를 사용헤 이를 직접 계산할 수 있다. aws nitro system은 일반적으로 non-nitro system 인스턴스 타입보다 더 많은 ip를 할당할 수 있다. vpc cni의 ip prefix 기능을 사용해 더 많은 ip를 사용할 수도 있다.
  - Version of the Amazon VPC CNI add-on that you’re running: vpc cni 버전마다 지원하는 인스턴스 타입이 다르다.
  - Whether you’re using security groups for Pods: security groups for pods를 모든 인스턴스 타입이 지원하는 것은 아니다.
- node health 정리
- vpc 요구 사항은 다음과 같다. ([VPC and subnet requirements](https://docs.aws.amazon.com/eks/latest/userguide/network-reqs.html#network-requirements-vpc))
  - vpc는 cluster, no, k8s resource를 위한 충분한 ip주소가 있어야 한다. 만약 vpc의 cidr보다 더 많은 ip가 필요할 경우 vpc에 cidr block을 추가할 수 있다. cluster 설정을 통해 cluster 생성 시 명시한 subnet과 security group을 변경 해 더 많은 ip를 사용할 수 있도록 할 수 있다. 하지만 cluster 생성 시 사용한 az 내에서만 변경 가능하다.
  - vpc의 DNS option(hostname, resoulution)이 모두 활성화되어야 한다. 그렇지 않으면 no가 cluster에 자신을 등록하지 못한다.
- eks 생성 시 명시하는 subnet을 cluster subnet이라고 한다. no가 cluster subnet에서 실행될 수도 있지만 권장하지 않는다. 왜냐하면 eks 업그레이드 간 추가 eks-managed eni가 생성되는데 이를 보장하기 위해서다. /28 cidr를 갖는 cluster subnet을 별도로 할당하는 것을 권장한다. ([VPC and Subnet Considerations](https://docs.aws.amazon.com/eks/latest/best-practices/subnets.html#_overview))
- cluster 생성 시 eks는 2-4개의 elastic network interface를 랜덤한 subnet에 자동 생성한다. cluster의 k8s 버전을 업데이트하는 경우 cluster 생성 시 만든 network interface를 삭제하고 동일 subnet 또는 다른 subnet에 새로운 network interface를 만든다. network interface가 생성되는 subnet을 제한하기 위해서는 2개의 subnet만 사용하면 된다. ([VPC and subnet requirements](https://docs.aws.amazon.com/eks/latest/userguide/network-reqs.html#network-requirements-vpc))
- cluster을 위한 subnet 요구 사항은 다음과 같다. ([VPC and subnet requirements](https://docs.aws.amazon.com/eks/latest/userguide/network-reqs.html#network-requirements-vpc))
  - 각 subnet은 최소 6개의 ip가 필요하지만 최소 16개를 권장한다.
  - 서로 다른 az에 존재하는 최소 2개의 subnet을 명시해야 한다.
  - public/private subnet 모두 가능하지만 private subnet을 권장한다.
- no를 위한 subnet 요구 사항은 다음과 같다. eks cluster 생성 시 명시한 subnet에 no, k8s resource를 배포하지 않아도 된다. 이 경우 eks는 해당 subnet에 elastic network interface를 생성하지는 않는다. ([VPC and subnet requirements](https://docs.aws.amazon.com/eks/latest/userguide/network-reqs.html#network-requirements-vpc))
  - no와 k8s resource 배포를 위한 충분한 ip 개수가 필요하다.
  - 인터넷 -> po inbound 접근이 필요한 경우 elb 배포를 위한 최소 1개의 public subnet이 필요하다. 가능하다면 no는 private subnet에 배포하는 것을 권장한다.
  - no를 public subnet에 배포하는 경우 subnet은 public ip를 할당하도록 설정돼야 한다.
  - no를 인터넷 outbound 접근이 불가능한 private subnet(nat 불가)에 배포하는 경우 no, po를 위한 vpc endpoint를 추가해야 한다(예를 들어 ecr, cloudwatch, sts, s3 등).
  - subnet에 elb를 배포하길 원하는 경우 private subnet에는 `kubernetes.io/role/internal-elb` key와 `1` value를 갖는 tag, public subnet에는 `kubernetes.io/role/elb` key와 `1` value를 갖는 tag를 추가해야 한다.
- eks cluster 생성 시 `eks-cluster-${EKS_NAME}-${UNIQUE_ID}` 이름을 갖는 cluster security group을 생성한다. 특징은 다음과 같다. ([Security group requirements](https://docs.aws.amazon.com/eks/latest/userguide/sec-group-reqs.html))
  - 아래 기본 inbound/outbound rule을 포함한다. inbound rule의 삭제할 경우 cluster 업데이트 시 자동으로 다시 생성한다.
    | Rule type | Protocol | Ports | Source | Destination                    |
    |-----------|----------|-------|--------|--------------------------------|
    | Inbound   | All      | All   | Self   |                                |
    | Outbound  | All      | All   |        | 0.0.0.0/0(IPv4) or ::/0 (IPv6) |
  - 3개의 기본 tag를 추가한다. 삭제할 경우 cluster 업데이트 시 자동으로 다시 생성한다.
  - eks는 cluster security group을 eks-managed eni, managed node group으로 관리되는 ec2 인스턴스 eni에 할당한다.
  - eks control plane과 managed node gorup의 no 사이 트래픽을 위한 inbound, outound rule이다. eks 생성 시 사용자 security group도 설정할 수 있는데 이 security group은 eks-managed eni에 할당되지만 node group의 ec2 인스턴스 eni에는 할당하지 않는다.
- cluster security group의 outbound rule을 더 제한적으로 설정하기 위해 기본 생성 규칙을 삭제하고 아래 필요한 최소 규칙만 추가할 수 있다.
  | Rule type      | Protocol    | Port  | Destination            |
  |----------------|-------------|-------|------------------------|
  | Outbound       | TCP         | 443   | Cluster security group |
  | Outbound       | TCP         | 10250 | Cluster security group |
  | Outbound (DNS) | TCP and UDP | 53    | Cluster security group |
  - 추가적으로 아래 규칙도 고려해야 한다.
    - no 간 통신을 위한 protocol과 port
    - no의 실행 시 cluster introspection, registration을 위해 접근하는 eks api에 대한 인터넷 접근 규칙
    - ecr, docker hub와 같은 image registry에 대한 접근 규칙
    - aws s3 접근
    - ipv4, ipv6 주소에 대해 별도의 규칙 관리 필요
- eks cluster 생성 시 `AmazonEKSClusterPolicy` policy를 permission으로 갖는 eks cluster iam role이 있어야 한다. ([Cluster IAM role](https://docs.aws.amazon.com/eks/latest/userguide/cluster-iam-role.html))
- eks ec2 instance를 node로 사용하는 경우 kubelet은 instance profile을 사용해 aws api를 호출해 얻은 정볼르 사용해 cluster에 join한다. node 생성 시 `AmazonEKSWorkerNodePolicy`, `AmazonEC2ContainerRegistryPullOnly` policy를 permission으로 갖는 eks node iam role이 있어야 한다. ([Node IAM role](https://docs.aws.amazon.com/eks/latest/userguide/create-node-role.html))

## 요약
- k8s의 autoscaling 옵션: pod hpa, vpa, cluster autoscaler, addon resizer
- linux container는 격리를 위해 namespace, cgroup(control group) 기술을 사용한다.
  - namespace: 프로세스가 독립적인 파일 시스템, 프로세스, 네트워크 인터페이스 등의 view를 가질 수 있도록 하는 기술
    - `mnt`: mount points
    - `user`: user and group ids
    - `uts(unix time sharing)`: hostname, nis domain name
    - `pid`: process ids
    - `network`: network devices, stacks, ports, etc
    - `ipc(inter process communication)`: system V IPC, POSIX message queues
    - `control groups`: control group root directory
  - cgroup: 프로세스가 사용할 수 있는 리소스(cpu, memory, 네트워크 I/O 등)을 제한하는 기술
- 이론적으로 container는 모든 리눅스 시스템에서 실행될 수 있지만 호스트의 커널을 사용하기 때문에 커널 버전에 영향을 받는다. 뿐만 아니라 특정 하드웨어 아키텍쳐용(x86, ARM 등)으로 만들어진 image는 해당 아키텍쳐에서만 실행될 수 있다.
- docker 기준 k8s는 po내 모든 container가 동일한 linux namespace를 공유하도록 docker를 설정한다. 파일 시스템의 경우 image에 저장되어 있어 기본적으로 완전히 분리된다.
  - 동일한 namespace: network, uts, ipc
  - 다른 namespace: uid, pid, mnt
- po의 manifest 중 spec.containers[\*].ports[\*].containerport는 container가 노출하는 포트 정보를 명시하기만 할 뿐 다른 기능은 없다. 이를 생략한다고 해서 포트를 통해 po에 연결할 수 있는 여부에 영향을 미치지 않는다.
- po 내 container는 app container, init container, sidecar container가 있다. app container는 `.spec.containers`, init container, sidecar container는 `.spec.initContainers` 필드를 사용해 정의한다. sidecar container는 `.spec.initContainers[].restartPolicy`가 Always인 init container다. `.spec.restartPolicy`는 app container, init container에만 영향을 미친다. `.spec.restartPolicy`가 OnFailure, Always일 경우 init container에게 모두 OnFailure로 적용된다.
- container 로그는 하루 단위, 10MB 크기 기준으로 롤링(rolling)된다.
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
- `kubectl delete --force --grace-period=0`: po의 immediate deletion
- `kuvectl apply --validate --dry-run`: client validation, dry run 수행(--validate=true가 기본값이기 때문에 flag를 명시하지 않아도 됨)
- `kubectl cluster-info`: control  plane의 주소(kube-apiserver)와 `kubernetes.io/cluster-service` label이 true인 service의 주소를 노출한다.
- `kubectl cluster-info dump --output-directory -A`: 클러스터의 전체 정보와 현재 ns, kube-system ns의 전체 정보를 stdout으로 출력한다.
- `kubectl cordon`: no를 unscheduleable 상태로 변경한다(no의 `.spec.unschedulable`을 false로 설정).
- `kubectl drain ${NODE}`:  no를 unscheduleable 상태로 변경하고, po를 eviction한다. 이 때 API-initiated eviction을 사용하는 것이 아니라 직접 po 목록을 대상을 갖고 delete한다([kubectl drain --help 명령어](https://kubernetes.io/images/docs/kubectl_drain.svg)).
- `kubectl taint`: no에 taint를 추가한다.

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
