k8s에서 ns는 단일 cluster 내에서 resource 집합에 대한 고립을 제공하는 메커니즘이다. 동일 ns 내에서 resource의 name은 고유해야한다. namespace-based scoping은 namespaced object에 대해서만 적용 가능하며 cluster-wide object에 대해서는 적용이 불가하다.

## When to Use Multiple Namespaces
ns는 멀티 유저 환경에서 resource를 나눈다.

Namespaces provide a scope for names. Names of resources need to be unique within a namespace, but not across namespaces. Namespaces cannot be nested inside one another and each Kubernetes resource can only be in one namespace.

ns는 [resource quota](https://kubernetes.io/docs/concepts/policy/resource-quotas/)을 사용해 여러 사용자 간 cluster resource를 나누는 방법이다.

동일한 소프트웨어의 버전과 같이 미세하게 다른 resource를 구분하는데 ns를 사용할 필요가 없다. 대신 label을 사용하면 된다.

> **Note**:  
> 운영 환경 cluster의 경우 default ns를 사용하지 않아야 한다.

## Initial namespaces
k8s는 기본적으로 4개의 ns를 제공한다.
- `default`: 사용자를 위한 기본 ns. 다른 어떤 ns에도 속하지 않을 경우 기본 ns
- `kube-node-lease`: 각 no와 관련된 [lease](https://kubernetes.io/docs/concepts/architecture/leases/) object를 관리하는 ns. node lease를 사용해 kubelet은 heartbeat을 전송하고 control plane은 no의 failure를 감지할 수 있다.
- `kube-public`: 이 ns는 모든 클라이언트(인증되지 않은 클라이언트 포함)가 read할 수 있다. 이 ns는 주로 cluster 사용을 위해 예약되며 클러스터 전체에서 일부 리소스를 공개적으로 볼 수 있고 읽을 수 있게 할 필요가 있는 경우에 사용된다. 이 ns의 공개적인 측면은 단순히 관례이며 필수 요구 사항은 
- `kube-system`: k8s system에 의해 생성되는 object의 ns

## Working with Namespaces
ns의 생성, 삭제는 [Admin Guide documentation for namespaces](https://kubernetes.io/docs/tasks/administer-cluster/namespaces)을 참고한다.

> **Note**  
> `kube-` 접두사를 갖는 ns는 k8s 시스템을 위해 예약됐기 때문에 이를 피해 사용하는 것을 권장한다.

### Viewing namespaces
kubectl get ns 명령어를 사용한다.

### Setting the namespace for a request
--namespace flag를 사용한다.

### Setting the namespace preference
```
kubectl config set-context --current --namespace=<insert-namespace-name-here>
# Validate it
kubectl config view --minify | grep namespace:
```

## Namespaces and DNS
svc를 생성할 때 svc는 관련 DNS entry를 생성한다. entry의 포맷은 `<service-name>.<namespace-name>.svc.cluster.local`이며 만약 container가 `<service-name>`만 사용하면 로컬 ns의 service로 해석한다. 이는 개발, 스테이징, 운영 환경에서 동일한 설정을 사용할 수 있기 때문에 유용하다. 만약 다른 ns에 접근하고자 할 경우에는 FQDN을 사용하면 된다.

그렇기 때문에 모든 ns의 이름은 [RFC 1123 DNS labels](https://kubernetes.io/docs/concepts/overview/working-with-objects/names/#dns-label-names)을 지켜야 한다.

> **Warning**  
> ns의 이름을 TLD(Top Level Domain)과 동일한 이름으로 생성하면 해당 ns의 svc는 public DNS 레코드를 덮어쓰는 짧은 이름의 DNS 이름을 가질 수 있다. "."으로 끝나지 않을 경우 DNS 룩업을 수행할 것이고 이는 public DNS보더 우선 순위가 높기 때문에 내부 svc ip정보를 응답한다.
> 
> 이러한 실수를 방지하기 위해 제한된 사용자만 ns를 만들 수 있도록 제한해야 한다. If required, you could additionally configure third-party security controls, such as [admission webhooks](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/), to block creating any namespace with the name of [public TLDs](https://data.iana.org/TLD/tlds-alpha-by-domain.txt).

## Not All Objects are in a Namespace
대부분의 k8s object는 특정 ns에 속한다. 하지만 ns 자체는 어떤 ns에도 속하지 않는다. 그리고 no, pv와 같은 low level resource는 어떤 ns에도 속하지 않는다.

ns 존재 유무는 kubectl api-resources 명령어로 조회 가능하다.

## Automatic labelling
k8s control plane은 모든 ns에 변경할 수 없는 `kubernetes.io/metadata.name` label key를 설정한다. 해당 label의 값은 ns의 이름이다.