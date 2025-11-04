k8를 multiple zone에서 운영하는 방법을 설명한다.

## Background
k8s는 여러 failure zone에 걸쳐 동작할 수 있도록 설계됐다. 일반적으로 zone은 region이라는 논리적 그룹에 속한다. 주요 cloud provider는 failure zone(또는 availability zone)의 집합을 region이라고 정의하며 일관된 기능을 제공한다. region 내에서 각 zone은 동일한 API와 서비스를 제공한다.

일반적인 cloud architecture는 한 zone의 장애가 다른 zone의 서비스를 손상시키는 것을 최소화하는 것을 목표로한다.

## Control plane behavior
All [control plane components](https://kubernetes.io/docs/concepts/overview/components/#control-plane-components) support running as a pool of interchangeable resources, replicated per component.

cluster control plane을 배포할 때 구성 요소의 replica를 여러 failure zone에 배치한다. 가용성이 중요한 문제라면 최소 3개 이상의 failure zone을 선택하고 각 개별 control plane 구성 요소(kube-apiserver, kube-scheduler, kube-controller-manager)를 최소 3개 이상의 failure zone에 각각 배포한다. 추가적으로 cloud-controller-manager를 실행하는 경우에도 모든 failure zone에 동일하게 배포해야 한다.

> **Note**:  
> k8s는 kube-apiserver에 대한 zone 간 회복성을 지원하지 않는다. 대신 dns round-robin, SRV record, 로드 밸런서를 사용해 가용성을 확보할 수 있다.

## Node behavior
k8s는 cluster의 여러 no에 workload resource(예를 들어 deploy, sts) po을 자동 배포한다. 이는 장애의 영향도를 줄여준다.

no가 시작되면 각 no의 kubelet은 no object에 label을 추가한다. 이러한 label은 [zone information](https://kubernetes.io/docs/reference/labels-annotations-taints/#topologykubernetesiozone)을 포함한다.

cluster가 여러 zone 또는 region에 있는 경우 no label와 [Pod topology spread constrains](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/)를 사용해 region, zone, 특정 no 간에 po의 분산 방법을 제어할 수 있다. kube-scheduler는 해당 정보를 사용해 po를 배치하며 이를 통해 전체 workload에 대한 영향도를 줄일 수 있다.

예를 들어 sts의 replica 3개가 모두 서로 다른 zone에서 실행되고 있는지 확인하는 제약조건을 설정할 수 있다. You can define this declaratively without explicitly defining which availability zones are in use for each workload.

### Distributing nodes across zones
k8s의 코어는 사용자를 위해 no를 생성하지 않는다. 사용자가 직접 no를 생성하거나 [Cluster API](https://cluster-api.sigs.k8s.io/) 같은 도구를 사용할 수 있다.

Using tools such as the Cluster API you can define sets of machines to run as worker nodes for your cluster across multiple failure domains, and rules to automatically heal the cluster in case of whole-zone service disruption.

## Manual zone assignment for Pods
po를 위한 [node selector constraints](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#nodeselector)을 사용할 수도 있다.

## Storage access for zones
pv가 생성되면 k8s는 zone과 관련된 모든 pv에 label을 자동 추가한다. 그리고 kube-scheduler는 NoVolumeZoneConflict 서술어(predicate)를 사용해 해당 pv에 대한 pvc를 갖는 po를 동일한 zone에 배치하는 것을 보장한다.

zone label을 추가하는 방법은 cloud provider, storage provisioner에 따라 달라질 수 있다. 정확한 설정을 위해서는 관련 문서를 참고해야한다.

kube-scheduler의 NoVolumeZoneConflict 로직은 pv의 label, `.spec.nodeAffinity` 정보를 참조해 po가 위치할 az에 존재하는 no에 po를 할당한다. pv의 az 정보를 나타내기위해 label, `.spec.nodeAffinity`을 이용할 수 있으며 해당 정보를 추가하는 것은 cloud provider, storage provisioner에 따라 다르다. aws-ebs-csi-driver의 경우 label 대신 `.spec.nodeAffinity` 필드를 사용해 az 정보를 나타낸다.

You can specify a StorageClass for PersistentVolumeClaims that specifies the failure domains (zones) that the storage in that class may use. To learn about configuring a StorageClass that is aware of failure domains or zones, see [Allowed topologies](https://kubernetes.io/docs/concepts/storage/storage-classes/#allowed-topologies).

## Networking
k8s는 zone-aware 네트워킹을 기능을 포함하지 않는다. [network plugin](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/)을 사용해 cluster 네트워킹을 구성할 수 있으며 해당 네트워크 솔루션은 zone과 관련된 구성 요소를 포함할 수도 있다. 예를 들어 cloud provider가 `type=LoadBalancer` svc를 지원하는 경우 lb는 동일한 zone으로 트래픽을 라우팅할 수 있다. 자세한 내용은 각 cloud provider의 공식 문서를 참고한다.

on-premise에서도 위와 같은 내용을 고려해야 한다. 다양한 failure zone 처리를 포함한 svc, ingress 동작은 cluster 설정에 따라 달라진다.

## Fault recovery
cluster를 구성할 때 한 region의 모든 failure zone이 동시에 오프라인 상태가 될 경우 서비스를 복원할 수 있는지 여부와 방법을 고려해야 할수도 있다. For example, do you rely on there being at least one node able to run Pods in a zone? Make sure that any cluster-critical repair work does not rely on there being at least one healthy node in your cluster. For example: if all nodes are unhealthy, you might need to run a repair Job with a special toleration so that the repair can complete enough to bring at least one node into service.

k8s는 이러한 문제에 대한 해답을 제시하지 않지만 사용자는 고려해야 할 사항이다.