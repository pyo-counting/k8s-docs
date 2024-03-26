클러스터는 k8s agent를 실행하는 no 집합이며 control plane이 관리한다. k8s v1.29는 최대 5,000개의 no로 구성된 cluster를 지원한다. 상세하게 k8s는 아래 규모를 지원한다.
- no 당 최대 110 po
- 총 no의 수는 5,000개
- 총 po의 수는 150,000개
- 총 container 수는 300,000개

no 추가 / 삭제를 통해 cluster를 스케일링할 수 있다. 스케일링 방법은 cluster 환경에 따라 다를 수 있다.

## Cloud provider resource quotas
많은 no가 있는 cluster를 생성할 때 각 cloud provider의 quota 이슈를 발생시키지 않도록 하기 위해 아래 내용을 고려해야 한다.
- cloud resource에 대한 quota 증가 요청
    - Computer instances
    - CPUs
    - Storage volumes
    - In-use IP addresses
    - Packet filtering rule sets
    - Number of load balancers
    - Network subnets
    - Log streams
- Gating the cluster scaling actions to bring up new nodes in batches, with a pause between batches, because some cloud providers rate limit the creation of new instances.

## Control plane components
대규모 cluster의 경우 충분한 computing, 기타 resource를 갖춘 control plane이 필요하다.

일반적으로 failure zone당 하나 또는 두 개의 control plane instance를 실행한다. 스케일링의 경우 먼저 vertical scaling을 수행하며 실패 지점에 도달했을 때는 horizontal scaling을 수행한다.

fault-tolerance을 제공하기 위해 failure zone 별로 적어도 하나의 instance를 실행해야 한다. k8s no는 동일한 failure zone에 있는 control plane 엔드포인트로 트래픽을 자동으로 라우팅하지 않지만 cloud provider는 이를 수행할 수 있는 고유한 메커니즘을 제공할 수 있다.

예를 들어 managed load balancer를 사용해 failure zone A의 kubelet과 po에서 발생하는 동일한 failure zone a A에 있는 control plane 호스트에만 전송하도록 설정할 수 있다. failure zone A의 단일 control plane 호스트 또는 엔드포인트가 오프라인 상태가 되면 zone A의 no에 대한 모든 control plane 트래픽이 zone 간 전송된다. 각 zone에 여러 개의 control plane 호스트를 실행하면 이러한 결과가 발생할 가능성이 낮아질 수 있다.

### etcd storage
대규모 cluster의 성능을 향상시키기 위해 Event object를 별도의 etcd 전용 instance에 저장할 수 있다.

cluster를 생성할 때 custom 도구를 사용해 아래 내용을 수행할 수 있다.
- 추가 etcd instancen을 실행 및 설정
- kube-apiserver가 해당 etcd에 event object를 저장할 수 있도록 설정

추가 내용은 [Operating etcd clusters for Kubernetes](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/), [Set up a High Availability etcd cluster with kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/setup-ha-etcd-with-kubeadm/)을 참고한다.

## Addon resources
k8s resource limit 기능은 po, container의 memory 누수로 인한 다른 구성 요소에 미칠 수 있는 영향을 최소화하는 데 도움을 줄 수 있다.

예를 들어 아래와 같이 logging 구성 요소에 CPU, memory limit을 설정할 수 있다.
``` yaml
  ...
  containers:
  - name: fluentd-cloud-logging
    image: fluent/fluentd-kubernetes-daemonset:v1
    resources:
      limits:
        cpu: 100m
        memory: 200Mi
```

addon의 기본 limit은 일반적으로 작은 크기의 cluster에서의 경험을 통해 수집한 데이터를 기반으로 한다. 대형 cluster에서 실행할 때는 기본 값보다 더 큰 resource를 소모한다. 규모가 큰 cluster를 운영할 때 이러한 기본 값을 사용하게되면 memory limit으로 계속해서 addon이 kill될 수도 있다.

그렇기 때문에 대형 cluster를 관리하기 위해 아래 내용을 고려한다.
- addon의 vertical scaling: addon에 대해 1개의 replica가 존재할 경우 request, limit을 확장한다.
- addon의 horizontal scaling: po의 replica를 증가시킨다. 물론 cluster가 매우 큰 경우 limit을 더 확장해야 한다. VerticalPodAutoscaler를 recommender mode로 사용해 request, limit에 대한 제안을 확인할 수도 있다.
- no당 1개의 po(ds으로 관리): cpu, memoy limit을 확장한다.