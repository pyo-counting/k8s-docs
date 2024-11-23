k8s cluster는 container화된 애플리케이션을 실행하는 no라고 불리는 worker machine의 집합이다. 모든 클러스터는 최소 한 개의 worker node를 가진다.

worker node는 애플리케이션의 구성요소인 po를 호스팅한다. control plane은 worker node와 클러스터 내 po를 관리한다. 운영 환경에서는 일반적으로 control plane이 여러 컴퓨터에서 실행되고, 클러스터는 일반적으로 여러 no를 실행하므로 fault-tolerance과 high availability를 제공한다.

![](https://kubernetes.io/images/docs/kubernetes-cluster-architecture.svg)

## Control Plane Components
control plane의 구성요소는 cluster와 관련된 전반적인 결정(예를 들어, 스케줄링)을 수행하고 cluster 이벤트(예를 들어, deploy replicas 필드가 충족되지 않을 경우 새로운 po를 구동시키는 것)를 감지하고 반응한다.

control plane 구성요소는 cluster 내 어떠한 머신에서든지 동작할 수 있다. 그러나 간결성을 위하여 보통 동일 머신 상에 모든 control plane 구성요소를 구동시키고, 사용자 container는 해당 머신 상에 동작시키지 않는다.

### kube-apiserver
API server는 k8s API를 노출하는 k8s control plane 구성요소다. API server는 k8s control plane의 프론트엔드다.

k8s API server의 주요 구현은 kube-apiserver다. kube-apiserver는 수평 확장이 가능하도록 디자인됐다. 즉, 더 많은 인스턴스를 배포해서 확장할 수 있다. 여러 kube-apiserver 인스턴스를 실행하고 인스턴스간의 트래픽을 균형있게 조절할 수 있다.

### etcd
모든 클러스터 데이터를 저장하는 k8s backing store로 사용되는 consistent, highly-available key value store.

k8s 클러스터에서 etcd를 backing store로 사용한다면 이 데이터에 대한 [backup](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/#backing-up-an-etcd-cluster) 계획은 필수이다.

etcd에 대한 자세한 정보는, 공식 [documentation](https://etcd.io/docs/)를 참고한다.

### kube-scheduler
no가 배정되지 않은 새로 생성된 po를 감지(watch)하고 실행할 no를 선택하는 conrol plane 구성요소다.

스케줄링 결정을 위해서 고려되는 요소에는 리소스에 대한 개별 또는 총 요구사항, 하드웨어/소프트웨어/정책 제약, affinity/anti-affinity, data locality, workload간 간섭, deadline 등을 포함한다.

### kube-controller-manager
컨트롤러 프로세스를 실행하는 control plane 구성요소다.

논리적으로 각 컨트롤러는 분리된 프로세스이지만 복잡성을 낮추기 위해 모두 단일 바이너리로 컴파일되고 단일 프로세스 내에서 실행된다.

컨트롤러의 예시는 다음과 같다.
- `node controller`: no가 다운되었을 때 통지와 대응에 관한 책임을 가진다.
- `job controller`: job object를 감시하며 해당 작업을 수행하기 위한 po를 생성한다.
- `endpointSlice controller`: EndpointSlice object를 관리한다(즉, svc와 po를 연결시킨다).
- `service account controller`: 새로운 ns에 대한 기본 sa을 생성한다.

위 controller 외에도 더 많은 controller가 존재한다.

### cloud-controller-manager
cloud 별 제어 로직을 내장한 k8s control plane 구성요소. [cloud-controller-manager](https://kubernetes.io/docs/concepts/architecture/cloud-controller/)를 사용하면 cluster를 cloud provider의 API에 연결하고 해당 cloud 플랫폼과 상호 작용하는 구성 요소를 cluster와만 상호 작용하는 구성 요소로 분리할 수 있다. cloud-controller-manager는 cloud provider와 관련된 controller만 실행한다. cloud 환경이 아니라면 해당 구성요소는 필요 없다.

kube-controller-manager와 마찬가지로 cloud-controller-manager는 논리적으로 독립적인 여러 제어 루프를 단일 바이너리로 결합한 단일 프로세스로 실행한다. 수평 확장도 가능하다.

아래 controller는 cloud provider 족속성을 가질 수 있다:
- `node controller`: 응답이 멈춘 이후 클라우드에서 no가 삭제됐는지 여부를 확인
- `route controller`: cloud 인프라 내 라우트 설정
- `service controller`: cloud provider load balancer 생성 / 업데이트 / 삭제

## Node Components
no 구성요소는 모든 no에서 실행되어 동작 중인 po를 관리하고 k8s runtime 환경을 제공한다.

### kubelet
cluster의 각 no에서 실행되는 에이전트. kubelet은 po에서 container 동작하도록 관리한다.

kubelet은 다양한 메커니즘을 통해 제공된 po 스펙(PodSpec)의 집합을 이용해 container가 해당 po 스펙에 따라 정상 동작하는 것을 보장한다. kubelet은 k8s를 통해 생성되지 않는 container는 관리하지 않는다.

### kube-proxy (optional)
kube-proxy는 cluster의 각 no에서 실행되는 네트워크 프록시로 k8s의 svc 개념의 구현부이다.

kube-proxy는 no의 네트워크 규칙을 유지 관리한다. 이 네트워크 규칙이 내부 네트워크 세션이나 cluster 바깥에서 po로 네트워크 통신을 할 수 있도록 해준다.

kube-proxy는 OS에 이용 가능한 패킷 필터링 계층이 있는 경우 이를 사용한다. 그렇지 않으면, kube-proxy는 트래픽 자체를 포워드(forward)한다.

kube-proxy와 동일한 기능을 수행하는 [network plugin](https://kubernetes.io/docs/concepts/architecture/#network-plugins)을 사용한다면 kube-rpoxy를 사용할 필요는 없다.

### Container runtime
container runtime은 container 실행을 담당하는 소프트웨어이다.

k8s는 containerd, CRI-O와 같은 container runtime, 모든 [Kubernetes CRI(Container Runtime Interface)](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-node/container-runtime-interface.md) 구현체를 지원한다.

## Addons
addon은 k8s resource(ds, deploy 등)를 이용하여 cluster 기능을 구현한다. 이들은 cluster 단위의 기능을 제공하기 때문에 addon에 대한 resource는 kube-system ns에 속한다.

사용 가능한 전체 확장 addon 리스트는 [Addons](https://kubernetes.io/docs/concepts/cluster-administration/addons/)을 참조한다.

### DNS
다른 addon들은 절대적으로 요구되지 않지만 대부분의 경우 k8s 클러스터는 [cluster DNS](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)를 갖추어야만 한다.

cluster DNS는 k8s svc를 위해 DNS 레코드를 제공해주는 DNS 서버다.

k8s에 의해 구동되는 container는 DNS search에서 이 DNS 서버를 자동으로 포함한다.

### Web UI (Dashboard)
[Dashboard](https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/) is a general purpose, web-based UI for Kubernetes clusters. It allows users to manage and troubleshoot applications running in the cluster, as well as the cluster itself.

### Container Resource Monitoring
[Container Resource Monitoring](https://kubernetes.io/docs/tasks/debug/debug-cluster/resource-usage-monitoring/)은 중앙 데이터베이스에 container에 대한 시계열 metrics를 기록하고 그 데이터를 조회하기 위한 UI를 제공한다.

### Cluster-level Logging
[cluster-level loging](https://kubernetes.io/docs/concepts/cluster-administration/logging/) 메커니즘은 검색/조회 인터페이스와 함께 중앙 로그 저장소에 container 로그를 저장한다.

### Network Plugins
[Network plugins](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/) are software components that implement the container network interface (CNI) specification. They are responsible for allocating IP addresses to pods and enabling them to communicate with each other within the cluster.

## Architecture variations
### Control plane deployment options
### Workload placement considerations
### Cluster management tools
### Customization and extensibility