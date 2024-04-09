분산 시스템에서는 공유 리소스를 lock하고 집합 구성원 간의 활동을 조정하는 메커니즘을 제공하는 lease가 필요한 경우가 있다. k8s에서 lease 개념은 no의 heartbeat, 구성 요소 사이에서 leader election과 같이 시스템의 중요한 기능에 coordination.k8s.io API Group의 [Lease](https://kubernetes.io/docs/reference/kubernetes-api/cluster-resources/lease-v1/) 리소스를 사용한다.

아래는 lease object 예제다.
``` sh
k get lease -A
NAMESPACE          NAME                                               HOLDER                                                                                                 AGE
argo-rollouts-ns   argo-rollouts-controller-lock                      kps-prd-plcc-helm-argo-rollouts-587bf87d48-pl2nn_837aa2e1-e4e6-431f-8b8c-616b9db424df                  7d1h
kube-node-lease    ip-172-30-49-111.ap-northeast-2.compute.internal   ip-172-30-49-111.ap-northeast-2.compute.internal                                                       7d5h
kube-node-lease    ip-172-30-62-229.ap-northeast-2.compute.internal   ip-172-30-62-229.ap-northeast-2.compute.internal                                                       7d5h
kube-system        apiserver-ia3e676y6id64oneb5y2mw45je               apiserver-ia3e676y6id64oneb5y2mw45je_125d9658-b87c-4667-9b7d-2eff7575c0c3                              8d
kube-system        apiserver-rcliy3icfb6lzuntbwqbu2j3iy               apiserver-rcliy3icfb6lzuntbwqbu2j3iy_12634b95-40d4-4cdc-9211-653bafa234d8                              8d
kube-system        aws-load-balancer-controller-leader                kps-prd-plcc-helm-aws-load-balancer-controller-5bf95cd79f-v6w49_a1eafd8e-0c31-4747-94cb-1d2b62714ff1   7d1h
kube-system        cloud-controller-manager                           ip-10-0-57-94.ap-northeast-2.compute.internal_6b1075f5-1057-4221-9f0e-2434d91f4eac                     8d
kube-system        cp-vpc-resource-controller                         ip-10-0-57-94.ap-northeast-2.compute.internal_44ca490b-56cd-4d51-aa2a-bc6573e62e92                     8d
kube-system        efs-csi-aws-com                                    1711798320015-382-efs-csi-aws-com                                                                      7d1h
kube-system        eks-certificates-controller                        ip-10-0-57-94.ap-northeast-2.compute.internal                                                          8d
kube-system        kube-controller-manager                            ip-10-0-57-94.ap-northeast-2.compute.internal_58e2ca46-9418-4fda-9ebb-2faec0ada0fd                     8d
kube-system        kube-scheduler                                     ip-10-0-57-94.ap-northeast-2.compute.internal_735b162b-c329-4d6a-a494-52d3de58af87                     8d
```

## Node heartbeats
k8s는 kube-apiserver로의 kubelet no heartbeat를 위해 Lease API를 사용한다. 모든 no은 kube-node-lease ns에 동일한 이름을 갖는 lease object가 있다. 기본적으로 모든 kubelet heartbeat는 lease object의 `.spec.renewTime` 필드를 업데이트하는 요청이다. k8s control plane은 no의 가용성을 결정하는데 해당 필드를 사용한다.

## Leader election
또한 k8s는 구성요소의 1개의 인스턴스만 실행되도록 보장한다. 이는 HA 구성에서 kube-controller-manager, kube-scheduler와 같은 control plane의 구성요소에서 사용하며 구성 요소의 인스턴스 하나만 활성화되고 다른 인스턴스는 대기상태여야 한다.

## API server identity
k8s v1.26부터 각 kube-apiserver는 자신의 식별 정보를 시스템의 나머지 부분에 발행하기 위해 Lease API를 사용합니다. 이것 자체로는 특별히 유용하지 않지만 클라이언트가 k8s control plane의 kube-apiserver 인스턴스 수를 확인할 수 있는 메커니즘을 제공한다. kube-apiserver lease의 존재는 각 kube-apiserver 간의 조정이 필요한 향후 기능을 가능하게 한다.

kube-apiserver가 소유한 lease object를 확인하기 위해 kube-system ns에서 `kube-apiserver-<sha256-hash>` 이름의 lease object를 확인하면 된다. 또는 apiserver.kubernetes.io/identity=kube-apiserver label selector를 사용할 수도 있다.

``` sh
kubectl -n kube-system get lease -l apiserver.kubernetes.io/identity=kube-apiserver
```

```
NAME                                        HOLDER                                                                           AGE
apiserver-07a5ea9b9b072c4a5f3d1c3702        apiserver-07a5ea9b9b072c4a5f3d1c3702_0c8914f7-0f35-440e-8676-7844977d3a05        5m33s
apiserver-7be9e061c59d368b3ddaf1376e        apiserver-7be9e061c59d368b3ddaf1376e_84f2a85d-37c1-4b14-b6b9-603e62e4896f        4m23s
apiserver-1dfef752bcb36637d2763d1868        apiserver-1dfef752bcb36637d2763d1868_c5ffa286-8a9a-45d4-91e7-61118ed58d2e        4m43s
```

lease 이름에 사용되는 SHA256 hash는 각 kube-apiserver의 서버 hostname을 기반으로 한다. 각 kube-apiserver는 cluster 내에서 고유한 hostname을 사용하도록 구성되어야 한다. 동일한 hostname을 사용하는 새로운 kube-apiserver 인스턴스는 새로운 식별자를 사용해 기존의 lease를 덮어쓰며 새로운 lease object를 생성하지 않는다. kube-apiserver가 사용하는 hostname을 확인하기 위해 `kubernetes.io/hostname` label을 확인한다.

``` sh
kubectl -n kube-system get lease apiserver-07a5ea9b9b072c4a5f3d1c3702 -o yaml
```

``` yaml
apiVersion: coordination.k8s.io/v1
kind: Lease
metadata:
  creationTimestamp: "2023-07-02T13:16:48Z"
  labels:
    apiserver.kubernetes.io/identity: kube-apiserver
    kubernetes.io/hostname: master-1
  name: apiserver-07a5ea9b9b072c4a5f3d1c3702
  namespace: kube-system
  resourceVersion: "334899"
  uid: 90870ab5-1ba9-4523-b215-e4d4e662acb1
spec:
  holderIdentity: apiserver-07a5ea9b9b072c4a5f3d1c3702_0c8914f7-0f35-440e-8676-7844977d3a05
  leaseDurationSeconds: 3600
  renewTime: "2023-07-04T21:58:48.065888Z"
```

더 이상 존재하지 않는 kube-apiserver의 만료된 lease는 새로운 kube-apivser에 의해 1시간 뒤 gc된다.

APIServerIdentity feature gate를 비활성화해서 kube-apiserver identity leases를 비활성화할 수 있다.

## Workloads
사용자는 workload에 대한 lease를 정의할 수도 있다. 예를 들어 peer가 아닌 leader만 작업을 수행하는 커스텀 controller를 실행할 수 있다. lease를 정의함으로써 controller replica는 coordination에 대한 k8s API를 이용해 leader를 선출할 수 있다. lease를 사용하기 위해서는 구성요소와 연결될 수 있도록 관련된 이름을 사용하는 것이 best practice다. 예를 들어, Foo라는 구성요소가 있는 경우 example-foo 이름을 갖는 lease를 사용한다.

cluster 관리자 또는 다른 엔드 유저가 구성요소의 여러 인스턴스를 배포할 수 있는 경우 이름 접두사를 선택하고 메커니즘(예를 들어 deploy 이름에 대한 해싱)을 선택해 lease의 이름 충돌을 피해야한다.

동일한 결과를 얻어낼 수 있는 경우 다른 방식을 사용할 수도 있다.