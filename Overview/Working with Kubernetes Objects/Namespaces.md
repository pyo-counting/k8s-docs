## Namespaces
k8s에서 ns는 단일 클러스터 내에서 resource 집합에 대한 고립을 제공하는 메커니즘이다. ns 내에서 resource의 name은 고유해야한다. namespace-based scoping은 namespaced object에 대해서만 적용 가능하며 cluster-wide object에 대해서는 적용이 불가하다.

### When to Use Multiple Namespaces
ns는 멀티 유저 환경에서 resource를 나눈다.

동일한 소프트웨어의 버전과 같이 미세하게 다른 resource를 구분하는데 ns를 사용할 필요가 없다. 대신 label을 사용하면 된다.

### Working with Namespaces
**Note**: kube- 접두사를 갖는 ns는 k8s 시스템이 사용하기 때문에 이를 피하는 것을 권장한다.

#### Viewing namespaces
k8s는 기본적으로 4개의 ns를 갖는다:

- default: 다른 어떤 ns에도 속하지 않을 경우 기본 ns
- kube-system: k8s system이 생성한 object가 속하는 ns
- kube-public: 자동으로 생성되며 모든 사용자가 조회 가능하다. 이는 전체 클러스터에서 공공으로 조회가 필요한 resource를 위해 예약됐다.
- kube-node-lease: 각 no의 lease object를 관리한다. Node leases allow the kubelet to send heartbeats so that the control plane can detect node failure.

#### Setting the namespace for a request

#### Setting the namespace preference

### Namespaces and DNS
svc를 생성할 때 svc는 관련 DNS entry를 생성한다. entry의 포맷은 \<service-name\>.\<namespace-name\>.svc.cluster.local이며 만약 container가 \<service-name\>만 사용하면 로컬 ns의 service로 해석한다. 이는 개발, 스테이징, 운영 환경에서 동일한 설정을 사용할 수 있기 때문에 유용하다. 만약 다른 ns에 접근하고자 할 경우에는 FQDN을 사용하면 된다.

**Warning**: By creating namespaces with the same name as public top-level domains, Services in these namespaces can have short DNS names that overlap with public DNS records. Workloads from any namespace performing a DNS lookup without a trailing dot will be redirected to those services, taking precedence over public DNS.

To mitigate this, limit privileges for creating namespaces to trusted users. If required, you could additionally configure third-party security controls, such as admission webhooks, to block creating any namespace with the name of public TLDs.

### Not All Objects are in a Namespace
대부분의 k8s object는 특정 ns에 속한다. 하지만 ns 자체는 어떤 ns에도 속하지 않는다. 그리고 no, psv와 같은 low level  resource는 어떤 ns에도 속하지 않는다.

ns 존재 유무는 kubectl api-resources 명령어로 조회 가능하다.

### Automatic labelling
k8s control plane은 NamespaceDefaultLabelName feature gate가 활성화 된 경우 모든 no에 변경 불가한 label kubernetes.io/metadata.name label을 추가한다.