k8s는 cloud-native 아키텍처를 기반으로하며 CNVF로부터 cloud native 정보 보안에 대한 모범 사례를 얻는다.

## Kubernetes security mechanisms
k8s는 여러 API, security 제어, 정보 보안 관리 방법의 일부를 위한 정책을 정의할 수 있다.

### Control plane protection
k8s의 주요 보안 메커니즘은 [control access to the Kubernetes API](https://kubernetes.io/docs/concepts/security/controlling-access)이다.

k8s control plane 내부, control plane과 client 사이에서 전송되는 데이터 암호화([data encryption in transit](https://kubernetes.io/docs/tasks/tls/managing-tls-in-a-cluster/))를 제공하기 위해 TLS를 사용하는 것을 권장한다. k8s control plane 내에 저장된 데이터에 대해 [encryption at rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/)을 활성화할 수 있다. this is separate from using encryption at rest for your own workloads' data, which might also be a good idea.

### Secrets
[Secret](https://kubernetes.io/docs/concepts/configuration/secret/) API는 비밀 정보의 설정 값을 보호할 수 있는 기본 기능을 제공한다.

### Workload protection
[Pod security standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)는 po와 container의 고립을 보장한다. 그리고 필요에 따라 [RuntimeClasses](https://kubernetes.io/docs/concepts/containers/runtime-class/)를 사용해 custom 고립을 사용할 수 있다.

[Network policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)는 po 사이 또는 po와 외부 사이의 네트워크 트래픽을 제어한다.

ecosystem의 보안 제어 프로젝트를 cluster에 배포해 po, container, image에 대한 보안을 구성할 수 있다.

### Auditing
k8s [audit logging](https://kubernetes.io/docs/tasks/debug/debug-cluster/audit/)은 cluster 내에서 작업 순서, 기록을 제공한다. cluster 사용자, k8s API를 사용하는 애플리케이션, control plane 자체에서 생성된 작업을 감시할 수 있다.

## Cloud provider security
> **Note**:  
> 아래 내용은 k8s 외부 provider를 참조한다. The Kubernetes project authors aren't responsible for those third-party products or projects. To add a vendor, product or project to this list, read the content guide before submitting a change. More information.

If you are running a Kubernetes cluster on your own hardware or a different cloud provider, consult your documentation for security best practices. Here are links to some of the popular cloud providers' security documentation:

## Policies
[NetworkPolicy](https://kubernetes.io/docs/concepts/services-networking/network-policies/)(네트워크 패킷 필터링에 대한 declarative 제어), [ValidatingAdmisisonPolicy](https://kubernetes.io/docs/reference/access-authn-authz/validating-admission-policy/)(k8s API를 사용해 변경할 수 있는 내용에 대한 declarative 제한)과 같은 k8s native 메커니즘을 사용해 보안 정책을 정의할 수 있다.

물론 k8s 주변의 넓은 ecosystem의 정책 구현에 의존할 수 있다. k8s는 이러한 ecosystem 프로젝트가 여러 정책 제어를 구현할 수 있도록 extension 메커니즘을 제공한다.

자세한 내용은 [Policies](https://kubernetes.io/docs/concepts/policy/)을 참고한다.