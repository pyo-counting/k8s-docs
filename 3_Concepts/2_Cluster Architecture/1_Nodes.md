k8s는 no에서 실행할 container를 po에 배치하여 workload를 실행한다. no는 cluster에 따라 가상 머신일 수도 있고 물리 머신일 수도 있다. 각 no는 control plane에 의해 관리되며 po를 실행하는 데 필요한 서비스를 포함한다.

일반적으로 cluster에는 여러 no가 존재한다. 테스트 환경에서는 1개의 no만 구성할 수도 있다.

모든 no의 공통 구성 요소는 `kubelet`, `container runtime`, `kube-proxy`가 있다.

## Management
kube-apiserver에 no를 추가하기 위한 두 가지 방법이 있다.
1. no의 kubelet이 control plane에 스스로 등록한다.
2. 사람이 직접 no object를 추가한다.

위 방법을 사용해 no를 등록하면 control plane은 새로운 no object가 유효한지 확인한다. 예를 들어 아래 JSON manifest를 통해 no를 생성할 경우
``` json
{
  "kind": "Node",
  "apiVersion": "v1",
  "metadata": {
    "name": "10.240.79.157",
    "labels": {
      "name": "my-first-k8s-node"
    }
  }
}
```

k8s는 내부적으로 no object를 생성한다. Kubernetes checks that a kubelet has registered to the API server that matches the `.metadata.name`   field of the Node. 만약 no가 healthy 상태라면(즉, 필요한 모든 서비스가 실행 중) po를 실행할 자격이 있다. healthy 상태가 아니라면 healthy 상태가 되기 전까지 해당 no는 cluster와 관련된 행동에서 제외된다.

> **Note**:  
> k8s는 유효하지 않은 no의 object를 보존하면서 healthy 상태가 될때까지 게속 체크한다.
> 
> health check를 멈추기 위해 no object를 직접 또는 controller가 삭제해야 한다.

no object의 이름은 DNS subdomain name 규칙을 따라야한다.

### Node name uniqeness
두 개의 no가 동시에 동일한 이름을 가질 수 없다. k8s는 동일한 이름의 resource에 대해 동일한 object로 생각한다. 동일한 이름을 갖는 no의 경우 동일한 state(네트워크, root disk 내용), label 등의 정보를 갖는다고 생각한다. 이로 인해 이름을 변경하지 않고 인스턴스가 수정되면 불일치가 발생할 수 있다. no를 교체하거나 업데이트해야 하는 경우 기존 no object를 먼저 kube-apiserver에서 제거하고 업데이투 후 다시 추가해야 한다.

### Self-registration of Nodes
kubelet 설정 파일 내 `.registerNode` 필드를 true(기본 값)으로 설정하면 kubelet은 kube-apiserver에 스스로 등록한다. 대부분의 경우 선호하는 방식이다.

스스로 등록할 경우 kubelet에 flag 또는 설정 파일 내 필드를 사용한다.
- `--kubeconfig`: kube-apiserver에 인증하기 위해 사용되는 credentials의 경로
- `--cloud-provider`: metadata를 읽기 위해 cloud provier와 통신하는 방법
- `.registerNode`: kube-apiserver에 스스로 등록할지 여부
- `.registerWithTaints`: no의 taints 목록(`<key>=<value>:<effect>`를 ,로 구분). `.registerNode`가 false일 경우 동작하지 않는다.
- `--node-ip`: (Optional) no의 ip 주소. no의 여러 ip주소를 사용할 수 있으며 dual-stack cluster의 경우 [configure IPv4/IPv6 dual stack](https://kubernetes.io/docs/concepts/services-networking/dual-stack/#configure-ipv4-ipv6-dual-stack)를 참고한다. 이 flag를 명시하지 않으면 no의 기본 ipv4 주소를 사용하고 ipv4 주소가 없으면 ipv6 주소를 사용한다.
- `--node-labels`: no의 label ([NodeRestriction admission plugin](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#noderestriction)에 의해 강제되는 label 규칙도 있다)
- `.nodeStatusUpdateFrequency`: (기본값 10s) kubelet이 no의 상태를 확인하는 주기. 만약 lease 기능이 활성화되지 않았을 때는 실제 no object의 `.status` 필드 업데이트까지 수행한다. 이 경우 kube-controller-manager의 `--node-monitor-grace-period` flag 값을 고려해야 한다.
- `.nodeStatusReportFrequency`: (기본값 5m) no의 상태 변화가 없을 경우 kubelet이 no object의 `.status` 필드를 업데이트하는 주기. kubelet은 no의 변화가 감지되면 해당 설정 값을 무시하고 바로 no object의 `.status` 필드를 업데이트를 수행한다. lease 기능이 활성화 됐을 때만 유효한 설정이다. But if `.nodeStatusUpdateFrequency` is set explicitly, `.nodeStatusReportFrequency`'s default value will be set to `.nodeStatusUpdateFrequency` for backward compatibility.
- `.nodeLeaseDurationSeconds`: (기본값 40) kubelet이 no의 lease object `.spec.renewTime`을 통해 no의 상태를 업데이트하는 주기. 해당 설정 값은 실제 시간을 나타내지 않으며 기본 값 40은 10s를 나타낸다. lease 업데이트가 실패하면 kubelet은 200ms를 시작으로 최대 7s까지의 지수 함수 backoff를 사용해 재시도를 수행한다.

[Node authorization mode](https://kubernetes.io/docs/reference/access-authn-authz/node/), [NodeRestriction admission plugin](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#noderestriction)가 활성화된 경우, kubelet은 자체 no의 resource만 생성/수정할 수 있는 권한이 있다. 

> **Note**:  
> no의 설정 변경이 필요한 경우 kube-apiserver에 no를 다시 등록하는 것이 권장 방법이다. 예를 들어 kubelet이 새로운 `--node-labels` flag를 사용해 재시작 되지만 동일한 no의 이름이 사용되는 경우, kube-apiserver에 대한 no 등록 시기에만 label이 설정되기 때문에 변경 사항이 적용되지 않는다.
> 
> kubelet 재시작 시 no의 설정이 변경되는 경우 해당 no에 스케줄링된 po는 오작동하거나 문제를 일으킬 수 있다. For example, already running Pod may be tainted against the new labels assigned to the Node, while other Pods, that are incompatible with that Pod will be scheduled based on this new label. Node re-registration ensures all Pods will be drained and properly re-scheduled.

### Manual Node administration
kubectl을 사용해 no object를 생성, 수정할 수 있다.

직접 no object를 생성하기 위해 kubelet 설정 파일 내 `.registerNode` 필드를 false로 설정한다.

`.registerNode` 필드와 상관없이 no obejct를 수정할 수 있다. 예를 들어 label 수정하거나 unschedulable(`.spec.unschedulable` 필드를 true로 설정))로 마킹할 수 있다.

no의 label은 po의 label selector와 같이 사용해 스케줄링을 제어할 수 있다.

no를 스케줄링 불가능하도록 만들면 kube-scheduler는 해당 no에 새로운 po를 스케줄링할 수 없지만 기존 po에는 영향을 미치지 않는다. 이는 no의 재부팅, 기타 유지 보수 준비 단계를 위해 유용하다.

no를 스케줄링 불가하게 하기 위해 아래 명령어를 사용할 수 있다.
``` bash
kubectl cordon $NODENAME
```

자세한 내용은 [Safely Drain a Node](https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/) 페이지를 참고한다.

> **Note**:  
> ds의 일부인 po는 스케줄링 불가한 no에서도 실행될 수 있다. ds는 일반적으로 workload 애플리케이션이 제거되더라도 no에서 실행되어야 하는 no lacal 서비스다.

## Node status
no의 `.status` 필드는 아래 정보를 포함한다.
- Addresses
- Conditions
- Capacity and Allocatable
- Info

no의 status 및 기타 정보를 조회하기 위해 kubectl 명령어를 사용할 수 있다.
``` bash
kubectl describe node <insert-node-name-here>
```

자세한 내용은 [Node Status](https://kubernetes.io/docs/reference/node/node-status/)를 참고한다.

## Node heartbeats
k8s no가 보내는 heartbeat는 cluster가 각 no의 가용성을 파악하고 failure가 감지되면 조치를 취할 수 있도록 도와준다.

2가지 형태의 heartbeat가 있다.
- no object의 `.status` 필드 업데이트
- kube-node-lease ns의 lease object. 각 no에 대해 lease object를 갖는다.

kubelet의 no heartbeat와 관련된 상세 사항은 [Node status](https://kubernetes.io/docs/reference/node/node-status/)를 참고한다.

## Node controller
node controller는 no의 다양한 측면을 관리하는 k8s control plane 구성요소다.

node controller는 no의 생명 주기 동안 여러 역할을 맡는다.
- no가 등록될 때 CIDR 블락을 할당한다(`--allocate-node-cidrs=true`일 경우). kube-controller-manager는 po 네트워킹을 위한 cluster의 CIDR 중 no CIDR를 각 no 별로 할당한다. CIDR 크기는 `--node-cidr-mask-size`을 통해 설정한다.
- controller의 내부 no 목록을 cloud provider의 사용 가능한 시스템 목록을 참고해 최신 상태로 유지하는 것이다. 클라우드 환경에서 실행할 때 no가 unhealthy 상태가 되면, node controller는 no에 대한 시스템이 이용 가능한지 cloud provider에 확인한다. 이용이 불가할 경우 node controller는 no 목록에서 해당 no를 삭제한다.
- no의 상태를 모니터링한다. node controller는 다음과 같은 책임이 있다.
    - no가 unreachable 상태가 될 경우, no의 .status 필드의 Ready condition을 업데이트 한다. 이 경우 node controller는 Ready condition을 `Unknown`으로 변경한다.
    - no가 unreachable(Unknown condition) 상태로 남아있는 경우, unreachable no에 있는 po를 eviction 하기 위해 `node.kubernetes.io/unreachable` taint key를 추가한다. k8s는 po에 명시적으로 `node.kubernetes.io/not-ready`, `node.kubernetes.io/unreachable` toleration key를 설정하지 않으면 해당 toleration과 `tolerationSeconds=300`을 설정하기 때문에 첫 eviction까지 5분을 기다린다. ([Taints based Evictions](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/#taint-based-evictions))
      > **Note**:  
      > 해당 페이지에서는 node controller가 pod eviction을 위해 API-initiated eviction(Eviction 리소스)을 사용한다고 되어있지만 [Taint based Evictions](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/#taint-based-evictions) 페이지를 확인해보면 node controller는 pod eviction을 위해 taint를 사용하는 것처럼 보인다.

기본적으로 node controller는 각 no의 상태를 5초 마다 확인한다. 이 주기는 kube-controller-manager 구성요소의 `--node-monitor-period` flag를 사용해 설정할 수 있다. node와 관련된 kube-controller-manager flag는 다음과 같다.
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

![](https://miro.medium.com/v2/resize:fit:720/format:webp/1*pvHnrsuXuGrOGrjq_OrKAA.jpeg)

위 그림에서는 `--pod-eviction-timeout`가 있지만 이는 k8s v1.29 기준 없어진 flag다. 해당 flag는 unhealthy no에서 po를 eviction하기까지 대기하는 시간이다. 대신 [Taints based Evictions](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/#taint-based-evictions)을 사용한다.([#39681](https://github.com/kubernetes/website/issues/39681))

### Rate limits on eviction
대부분의 경우 node controller는 초당 eviction 비율을 `--node-eviction-rate`(기본값 0.1)로 제한한다. 즉, 10초당 1개의 no에서만 po를 제거한다.

availability zone의 no가 unhealthy 상태가 되면 no eviction 동작은 바뀐다. node controller는 동시에 availability zone에서 unhealthy 상태인 no의 비율(Ready condition이 Unknown 또는 False)을 확인한다.
- unhealthy no의 비율이 적어도 `--unhealthy-zone-threshold`(기본값 0.55)면 eviction rate가 감소한다.
- cluster의 규모가 작은 경우(즉, `--large-cluster-size-threshold`(기본값 50) 이하의 no개수), eviction이 중지된다.
- 위 모든 경우에 해당하지 않으면 eviction 비율이 `--secondary-node-eviction-rate`(기본값 0.01)로 줄어든다.

이러한 정책이 availability zone 마다 적용되는 이유는 한 availability zone이 control plane에서 분리되는 경우 다른 availability zone은 연결된 상태로 유지될 수 있기 때문이다. cluster가 여러 cloud provider의 availability zone에 걸쳐 있지 않으면 eviction 메커니즘은 availability zone당 불가용성은 고려하지 않는다.

no를 여러 availability zone에 분산하는 주요 이유 중 하나는 한 zone 전체가 다운될 때 workload를 healthy zone으로 이동할 수 있도록 하기 위함이다. 따라서 한 zone의 모든 no가 unhealthy 상태가 되면 node controller는 evection 비율을 `--node-eviction-rate` 값으로 사용한다. 모든 zone이 unhealthy(cluster의 모든 no가 unhealthy) 상태가 되면 control plane과 node 간 연결에 대한 문제가 있는 것으로 간주하고 eviction을 수행하지 않는다. 일부 no가 다시 나타나면 node controller는 남아있는 unhealthy, unreachable no에서 po를 eviction한다.

그리고 node controller는 NoExecute taint가 있는 no에서 실행되는 po를 eviction하는 책임이 있다. 단, 해당 po가 해당 taint에 대한 toleration이 없을 경우에만 해당한다. node controller는 no가 unreachable, ready가 아닌 no에 대해 해당 taint를 추가한다. 이를 통해 kube-scheduler가 해당 no에 po를 배치하지 않도록한다.

## Resource capacity tracking
no object는 no의 리소스 capacity에 대한 정보를 추적한다: 예를 들어 이용 가능한 메모리와 CPU 정보. kubelet을 이용한 no의 self register는 등록 시 capacity에 대한 정보를 제공한다. 반대로 직접 no를 추가할 경우 용량 정보를 설정해야 한다.

kube-scheduler는 no에 실행 중인 po에 대한 충분한 리소스가 있음을 보장한다. kube-scheduler는 no에 존재하는 container의 resource request에 대한 총합이 no의 capacity보다 크지 않음을 확인한다. request의 총합은 kubelet에 의해 관리되는 모든 container를 포함하며 container runtime을 통해 직접 실행된 container와 kubelet의 제어 외의 프로세스는 제외한다.

> **Note**:  
> po로 배포되지 않은 프로세스에 대한 resource를 미리 예약하기 원할 경우 [reserve resources for system daemons](https://kubernetes.io/docs/tasks/administer-cluster/reserve-compute-resources/#system-reserved) 페이지를 참고한다.

## Node topology
kubelet에 TopologyManager [feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)를 활성화한 경우 kubelet은 리소스 할당 결정을 할 때 topology 힌트를 이용할 수 있다. 관련해 [Control Topology Management Policies on a Node](https://kubernetes.io/docs/tasks/administer-cluster/topology-manager/) 페이지를 참고한다.

## Swap memory management
no에 swap을 활성화하기 위해 kubelet의 `NodeSwap` feature gate 활성화(기본 값 true), kubelet의 `.failSwapOn`이 false(기본 값 true)어야한다. po가 swap을 사용하기 위해서는 kubelet의 `.swapBehavior`이 NoSwap (기본 값)이면 안된다.