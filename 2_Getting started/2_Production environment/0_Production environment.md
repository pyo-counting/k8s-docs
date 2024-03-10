## Production considerations
## Production cluster setup
## Production control plane
## Production worker nodes
운영 환경에 알맞는 품질의 workload는 탄력적(resilient)이어야 하며, workload가 의존하는 모든 것(예를 들어 CoreDNS)들도 탄력적이어야 한다. control plane은 cloud provider가 관리할 수 있지만 worker node는 여전히 직접 관리가 필요하다.
- node 설정: node는 물리적/가상 머신일 수 있다. 직접 node를 설치 및 관리하기 위해 적절한 [node component](https://kubernetes.io/docs/concepts/overview/components/#node-components) 설치할 수 있다.
    - 적절한 memory, cpu, disk 속도, storage 용량 고려 필요
    - Whether generic computer systems will do or you have workloads that need GPU processors, Windows nodes, or VM isolation.
- 유효한 node: kubernets cluster에 참여하기 위한 필수 사항을 충족했는지 확인하기 위한 정보는 [Valid node setup](https://kubernetes.io/docs/setup/best-practices/node-conformance/)를 참고한다.
- cluster에 node 추가: cluster에 node를 추가하는 방법은 [https://kubernetes.io/docs/concepts/architecture/nodes/](https://kubernetes.io/docs/concepts/architecture/nodes/)를 참고한다.
- scale node: [Considerations for large clusters](https://kubernetes.io/docs/setup/best-practices/cluster-large/)를 참고해 pod, container의 개수에 따른 node의 개수를 고려한다.
- autoscale node: 대부분의 cloud provier는 unhealthy node 대체, node의 scale을 위한 [Cluster Autoscaler](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler#readme)를 제공한다. [Frequently Asked Questions](https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/FAQ.md)에서 autoscaler가 어떻게 동작하는 지 확인할 수 있고, [Deployment](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler#deployment)에서 cloud provider에 따라 어떻게 구현됐는지 확인할 수 있다. For on-premises, there are some virtualization platforms that can be scripted to spin up new nodes based on demand.
- node health check 설정: 중요 workload의 경우 node, pod가 정상적으로 실행 중인지 중요하다. [Node Problem Detector](https://kubernetes.io/docs/tasks/debug/debug-cluster/monitor-node-health/) daemon을 사용해, node가 정상인지 확인할 수 있다.

## Production user management
운영 환경에서는 수 십 ~ 수 백명의 사용자가 cluster에 접근하는 환경이다. 테스트 환경에서는 1개의 관리 계정만 사용할 수 있다. 하지만 운영 환경에서는 서로 다른 namespace에 대해 접근 권한을 관리할 필요가 있을 수 있다.

운영 환경에서는 cluster에 접근하는 사람의 신원을 확인(authentication)하고 요청하는 작업을 수행할 수 있는 권한이 있는지 여부를 결정(authorization)하는 전략을 잘 선택해야 한다.

- authentication: apiserver는 클라이언트 인증서, bearer token, 인즌 proxy 또는 HTTP basic auth를 사용해 사용자를 인증할 수 있다. plugin을 사용해 apiserver는 LDAP, Kerberos와 같은 조직에서 사용하던 기존 인증 방법을 활용할 수도 있다. kubernetes 사용자 인증을 위한 자세한 내용은 [Authentication](https://kubernetes.io/docs/reference/access-authn-authz/authentication/)를 참고한다.
- authorization: 일반 사용자에게 권한을 부여하기 위해 RBAC, ABAC 중 선택할 수 있다. 사용자 계정을 승인하기 위한 각 방법에 대해서는 [Authorization Overview](https://kubernetes.io/docs/reference/access-authn-authz/authorization/)를 참고한다.
    - [RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/): 인증된 사용자에게 특정 권한 집합을 허용해 cluster에 대한 접근을 제어할 수 있다. 특정 namespace(Role)) 또는 전체 cluster(ClusterRole)에 대한 권한을 할당할 수 있다. 그리고 RoleBindings, ClusterRoleBindings을 사용해 해당 권한을 특정 사용자에게 할당할 수 있다.
    - [ABAC](https://kubernetes.io/docs/reference/access-authn-authz/abac/): cluster의 리소스 특성을 기반으로 정책을 만들고 해당 속성을 기반으로 접근을 허용하거나 거부할 수 있다. 정책 파일의 각 라인은 subject(user 또는 group), 리소스 특성, 비-리소스 특성(/version 또는 /api), readonly, versioning 특성(apiVersion, Kind), spec을 식별한다.

운영 환경 kubernetes cluster에 대해 인증, 권한을 설정할 때 고려할 사항은 다음과 같다.
- authorization mode 설정: Kubernetes API server(kube-apiserver)의 `--authorization-mode` flag를 사용해 authorization mode를 설정해야 한다. 예를 들어, /etc/kubernetes/manifests/kube-adminserver.yaml 파일에서 해당 flag를 Node, RABC로 설정할 수 있다.
- 클라이언트 인증서, role binings(RBAC) 만들기: RBAC authorization을 사용하면 사용자는 cluster CA에 의해 서명될 수 있는 CSR(Certificate Signing Request)를 생성할 수 있다. 그리고 각 사용자에게 Role, ClusterRoles를 할당할 수 있다. 자세한 내용은 [Certificate Signing Requests](https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests/)를 참고한다.
- ABAC:
- admission controller 고려:

## Set limits on workload resources
## What's next