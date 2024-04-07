운영 환경 k8s는 계획과 준비가 필요하다. 만약 중요한 workload를 실행한다면 회복력이 있게 설정되어야 한다. 이 페이지에서는 운영 환경에서 실행할 준비가 될 수 있도록 cluster를 설정하는 방법을 설명한다.

## Production considerations
일반적으로 운영 환경 k8s는 개발, 테스트 환경보다 추가 필요 사항이 요구된다. 운영 환경은 많은 사용자 들의 접근, 일관된 가용성(consistent availability), 요구 사항 변경을 수용할 수 있는 resource가 필요하다.

운영 환경 k8s을 구현할 장소(on premise 또는 cloud)를 결정할 때 k8s cluster에 대한 요구 사항은 아래 내용에 영향을 받는다.
- 가용성(availability): 단일 k8s는 단일 실패 지점(single point of failure)을 갖는다. 고가용성 클러스터는 다음을 의미한다.
    - worker node와 control plane을 분리한다.
    - 여러 node에 control plane 구성 요소를 구성한다.
    - kube-apiserver에 대한 트래픽을 로드 밸런싱한다.
    - worker node의 가용성을 보장한다.
- 확장성(scale): 안정적인 요구 사항이 필요하다면 이에 맞도록 구축하면 된다. 하지만 예상된 요구 사항이 시간에 따라 변할 수 있기 때문에 control plane, worker node에 대한 확장성을 계획해야 한다.
- 보안과 접근 제어(security and management): 테스트 환경에서는 k8s에 대한 관리자 권한을 사용하는 것이 일반적이다. 하지만 중요한 workload가 있는 공유 cluster에서는 각 사용자의 접근을 제어해야 한다. 이를 위해 rbac과 같은 보안 메커니즘을 사용해야 한다. [policy](https://kubernetes.io/docs/concepts/policy/), [container resource](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)를 사용해 사용자와 workload가 접근할 수 있는 resource을 제한할 수 있다.

직접 k8s 운영 환경을 구축하기 전에 turnkey cloud solution의 도입을 고려한다. 이러한 솔루션은 아래와 같은 이점을 갖는다.
- 서버리스(serverless): cluster를 관리하지 않고 사용자는 workload만 실행하면 된다. cpu, 메모리, 디스크 사용 요청에 대한 비용을 지불해야 한다.
- 관리되는 control plane: provider가 cluster의 control plane의 확장, 가용성을 관리하고 패치, 업그레이드를 관리한다.
- 관리되는 worker node: 필요에 맞게 node pool을 구성하면 provider가 해당 node를 사용할 수 있는지 확인하고 필요할 때 이용한다.
- integration: storage, container registry, auhentication과 같이 k8s 운영에 필요한 서비스를 통합한 provier도 존재한다.

Whether you build a production Kubernetes cluster yourself or work with partners, review the following sections to evaluate your needs as they relate to your cluster’s control plane, worker nodes, user access, and workload resources.

## Production cluster setup
control plane은 여러 work node에서 실행되는 각기 다른 workload를 관리한다. 물론 worker node는 k8s의 pod를 실행하기 위한 단일 개체를 나타낼 뿐이다.

### Production control plane
가장 간단한 k8s cluster는 모든 control plane, worker node를 동일 서버에서 실행한다. [Kubernetes Components](https://kubernetes.io/docs/concepts/overview/components/)에 표현된 것 같이 worker node를 추가해서 cluster를 확장할 수 있다.

하지만 고가용성 cluster를 제공하기 위해 control plane에 대한 확장도 고려해야 한다. 단일 서버에서 control plane을 운영하는 것은 고가용성을 제공하지 못한다. 고가용성 제공을 위해 아래 내용을 고려해야 한다.
- 배포 도구 선택: kubeadm, kops, kubespray와 같은 도구를 사용해 control plane을 배포할 수 있다. 이러한 도구를 이용한 배포는 [Installing Kubernetes with deployment tools](https://kubernetes.io/docs/setup/production-environment/tools/)를 참고한다. 물론 원하는 [Container Runtimes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)를 사용할 수 있다.
- 인증서 관리: control plane 서비스 간 안정적인 통신은 인증서를 통해 이루어진다. 인증서는 배포 도구에서 자동적으로 생성될 수 있으며 사용자의 인증서를 사용할 수도 있다. 자세한 내용은 [PKI certificates and requirements](https://kubernetes.io/docs/setup/best-practices/certificates/)를 참고한다.
- kube-apiserver를 위한 로드 밸런서 설정: 로드 밸런서를 구성해 외부 API 요청을 각 서버의 kube-apiserver로 분산한다. 자세한 내용은 [Create an External Load Balancer](https://kubernetes.io/docs/tasks/access-application-cluster/create-external-load-balancer/)를 참고한다.
- etcd 서비스의 분리와 백업: etcd 서비스는 다른 control plane 구성요소와 동일한 시스템에서 실행하거나 보안과 가용성을 강화하기 위한 별도의 시스템에서 실행할 수 있다. etcd는 cluster의 설정 데이터를 저장하기 때문에 필요할 때 데이터베이스를 복구할 수 있도록 etcd 데이터베이스를 정기적으로 백업해야 한다. 자세한 내용은 [etcd FAQ](https://etcd.io/docs/v3.5/faq/), [Operating etcd clusters for Kubernetes](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/), [Set up a High Availability etcd cluster with kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/setup-ha-etcd-with-kubeadm/)을 참고한다.
- 여러 control plane 시스템 생성: 고가용성을 위해 control plane은 단일 서버에 제한되면 안된다. control plane이 init 서비스(또는 systemd)로 실행되는 경우 최소 3대 이상의 서버에서 실행되어야 한다. However, running control plane services as pods in Kubernetes ensures that the replicated number of services that you request will always be available. scheduler는 fault tolerence를 보장해야 하지만 고가용성을 제공하지 않아야 한다. 일부 배포 도구는 k8s 서비스의 리더를 선출하기 위해 Raft 합의 알고리즘을 사용한다. primary가 없어지면 다른 service가 다시 리더로 선출된다.
- multiple zone에 대한 분배: 항상 cluster의 고가용성을 제공하기 위해 여러 data center(cloud 환경에서는 zone)에서 운영되는 cluster 생성을 고려한다. zone의 집합은 region이라고 부른다. cluster를 동일 region의 여러 zone에 분산하면 한 zone의 장애에도 고가용성을 보장할 수 있다. 자세한 내용은 [Running in multiple zones](https://kubernetes.io/docs/setup/best-practices/multiple-zones/)을 참고한다.
- 지속적인 기능 관리: 클러스터의 보안, 상태를 지속적 관리해야 한다. 예를 들어 kubeadm를 사용하는 경우 [Certificate Management](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-certs/), [Upgrading kubeadm clusters](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)을 참고한다. cluster 관련 자세한 내용은 [Administer a Cluster](https://kubernetes.io/docs/tasks/administer-cluster/)을 참고한다.

To learn about available options when you run control plane services, see kube-apiserver, kube-controller-manager, and kube-scheduler component pages. 고가용성을 제공하는 control plane을 위해서[ Options for Highly Available topology](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/ha-topology/), [Creating Highly Available clusters with kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/), [Operating etcd clusters for Kubernetes](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/)을 참고한다. etcd 백업 계획은 [Backing up an etcd cluster](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/#backing-up-an-etcd-cluster)을 참고한다.

### Production worker nodes
운영 환경에 알맞는 품질의 workload는 탄력적(resilient)이어야 하며, workload가 의존하는 모든 것(예를 들어 CoreDNS)들도 탄력적이어야 한다. control plane은 cloud provider가 관리할 수 있지만 worker node는 여전히 직접 관리가 필요하다.
- node 설정: node는 물리적/가상 머신일 수 있다. 직접 node를 설치 및 관리하기 위해서는 적절한 [Node services](https://kubernetes.io/docs/concepts/overview/components/#node-components) 설치해야 한다.
    - 적절한 memory, cpu, disk 속도, storage 용량 고려 필요
    - Whether generic computer systems will do or you have workloads that need GPU processors, Windows nodes, or VM isolation.
- 유효한 node: k8s cluster에 참여(join)하기 위한 요구 사항을 충족했는지 확인하기 위한 정보는 [Valid node setup](https://kubernetes.io/docs/setup/best-practices/node-conformance/)를 참고한다.
- cluster에 node 추가: cluster에 node를 추가하는 방법은 [Nodes](https://kubernetes.io/docs/concepts/architecture/nodes/)를 참고한다.
- scale node: [Considerations for large clusters](https://kubernetes.io/docs/setup/best-practices/cluster-large/)를 참고해 pod, container의 개수에 따른 node의 개수를 고려한다.
- autoscale node: 대부분의 cloud provier는 unhealthy node 대체, node의 scale을 위한 [Cluster Autoscaler](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler#readme)를 제공한다. [Frequently Asked Questions](https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/FAQ.md)에서 autoscaler가 어떻게 동작하는지 확인할 수 있고, [Deployment](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler#deployment)에서 각 cloud provider가 제공하는 솔루션 목록을 확인할 수 있다. For on-premises, there are some virtualization platforms that can be scripted to spin up new nodes based on demand.
- node health check 설정: 중요 workload의 경우 node, pod가 정상적으로 실행 중인지 중요하다. [Node Problem Detector](https://kubernetes.io/docs/tasks/debug/debug-cluster/monitor-node-health/) daemon을 사용해, node가 정상인지 확인할 수 있다.

## Production user management
운영 환경에서는 수 십 ~ 수 백명의 사용자가 cluster에 접근하는 환경이다. 테스트 환경에서는 보통 관리자 계정 1개만 사용한다. 하지만 운영 환경에서는 서로 다른 namespace에 대해 접근 권한을 관리할 필요가 있을 수 있다.

운영 환경에서는 cluster에 접근하는 사람의 신원을 확인(authentication)하고 요청하는 작업을 수행할 수 있는 권한이 있는지 여부를 결정(authorization)하는 전략을 잘 선택해야 한다.
- authentication: kube-apiserver는 클라이언트 인증서, bearer token, 인증 proxy 또는 HTTP basic auth를 사용해 사용자를 인증할 수 있다. plugin을 사용해 kube-apiserver는 LDAP, Kerberos와 같은 조직에서 사용하던 기존 인증 방법을 활용할 수도 있다. kubernetes 사용자 인증을 위한 자세한 내용은 [Authentication](https://kubernetes.io/docs/reference/access-authn-authz/authentication/)를 참고한다.
- authorization: 일반 사용자에게 권한을 부여하기 위해 RBAC, ABAC 중 선택할 수 있다. 두 방식의 사용자 계정을 승인 방법에 대해서는 [Authorization Overview](https://kubernetes.io/docs/reference/access-authn-authz/authorization/)를 참고한다.
    - [RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/): 인증된 사용자에게 특정 권한 집합을 허용해 cluster에 대한 접근을 제어할 수 있다. 특정 namespace(Role) 또는 전체 cluster(ClusterRole)에 대한 권한을 할당할 수 있다. 그리고 RoleBindings, ClusterRoleBindings을 사용해 해당 권한을 특정 사용자에게 할당할 수 있다.
    - [ABAC](https://kubernetes.io/docs/reference/access-authn-authz/abac/): cluster의 리소스 특성을 기반으로 정책을 만들고 해당 속성을 기반으로 접근을 허용하거나 거부할 수 있다. 정책 파일의 각 라인은 subject(user 또는 group), 리소스 특성, 비-리소스 특성(/version 또는 /api), readonly, versioning 특성(apiVersion, Kind), spec을 식별한다. [Examples](https://kubernetes.io/docs/reference/access-authn-authz/abac/#examples)을 참고한다.

운영 환경 k8s cluster에 대해 인증, 권한을 설정할 때 고려할 사항은 다음과 같다.
- authorization mode 설정: kube-apiserver의 `--authorization-mode` flag를 사용해 authorization mode를 설정해야 한다. 예를 들어, /etc/kubernetes/manifests/kube-adminserver.yaml 파일에서 해당 flag를 Node, RABC로 설정할 수 있다.
- 클라이언트 인증서, role binings(RBAC) 생성: RBAC authorization을 사용하면 사용자는 cluster CA에 의해 서명될 수 있는 CSR(Certificate Signing Request)를 생성할 수 있다. 그리고 각 사용자에게 Role, ClusterRoles를 할당할 수 있다. 자세한 내용은 [Certificate Signing Requests](https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests/)를 참고한다.
- ABAC:
- admission controller 고려: kube-apiserver에 들어오는 요청에 대한 추가적인 인가 방식은 [Webhook Token Authentication](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#webhook-token-authentication)이 있다. kube-apiserver에 [Admission Controller](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/)을 추가해 webhook, 특수 인가 유형을 활성화해야 한다.

## Set limits on workload resources
workload는 k8s control plane 외부, 내부 모두에 영향을 줄 수 있다. cluster workload의 요구 사항에 맞게 아래 내용을 고려해야 한다.
- ns의 제한 설정: memory, cpu와 같은 resource에 대해 ns 마다 quota 설정을 한다. 자세한 내용은 [Mnage Memory, Cpu, and API Resources](https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/)을 참고한다. 제한을 상속 받기 위해 [Hierarchical Namespaces](https://kubernetes.io/blog/2020/08/14/introducing-hierarchical-namespaces/)를 고려할 수 있다.
- dns 요구 사항에 대한 준비: worklaod의 scale up에 대해서 DNS service 역시 scale up해야 한다. 자세한 내용은 [Autoscale the DNS service in a Cluster](https://kubernetes.io/docs/tasks/administer-cluster/dns-horizontal-autoscaling/)를 참고한다.
- 추가적인 sa 생성: 사용자 계정은 사용자가 cluster에서 무엇을 할 수 있는지 결정하고 sa는 특정 ns에서 po의 접근을 정의한다. 기본적으로 po는 ns의 기본 sa를 사용한다. 자세한 내용은 [Managing Service Account](https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/)를 참고한다.
    - 특정 container registry에서 image를 pull할 수 있도록 secret을 추가한다. 자세한 내용은 [Configure Service Accounts for Pods](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/)를 참고한다.
    - sa에 RBAC을 할당한다. 자세한 내용은 [ServiceAccount permissions](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#service-account-permissions)을 참고한다.

## What's next