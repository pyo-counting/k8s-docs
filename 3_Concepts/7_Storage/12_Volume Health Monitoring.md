CSI volume health monitoring을 통해 CSI driver가 실제 storage system의 비정상적인 volume 상태를 감지해 pvc, po의 event 객체로 보고할 수 있다.

## Volume health monitoring
k8s volume health monitoring은 k8s가 CSI를 구현하는 방법의 일부다. volume health monitoring 기능은 두 가지 구성 요소로 구현된다: External Health Monitor controller, kubelet

CSI Driver가 controller으로부터 volume health monitoring를 지원하는 경우, CSI volume에 비정상적인 volume 상태가 감지될 때 관련 pvc에 event 객체가 보고된다.

CSI Driver가 node로부터 volume health monitoring를 지원하는 경우, CSI volume에 비정상적인 volume 상태가 감지될 때 pvc를 사용하는 모든 po에 event 객체가 보고된다. 게다가, volume health 정보가 kubelet의 VolumeStats metrics로 노출된다. 새로운 kubelet_volume_stats_health_status_abnormal metric이 존재한다. 해당 metric은 2개 label(namespace, persitentvolumeclaim)을 갖는다. 해당 metric의 값은 1 또는 0이다. 1은 volume이 unhealthy, 1은 volume이 healthy를 의미한다.

**Note**: node 내 해당 기능을 사용하기 위해 CSIVolumeHealth feature gate를 활성화해야 한다.