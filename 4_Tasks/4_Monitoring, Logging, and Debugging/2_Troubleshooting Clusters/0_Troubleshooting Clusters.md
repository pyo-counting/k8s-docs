## Listing your cluster
클러스터에서 디버그를 위해 가장 먼저 할 일은 no가 올바르게 등록되었는지 확인하는 것이다:

``` bash
kubectl get no
```

출력 결과를 통해 모든 예상되는 no가 모두 조회되는지와 모두 Ready state인지 확인한다.

cluster의 전체적인 상태 확인을 위해서는 아래 명령어를 실행한다:

``` bash
kubectl cluster-info dump
```

### Example: debugging a down/unreachable node
디버깅을 할 때 no의 상태를 확인하는 것이 유용할 수 있다. po와 마찬가지로 `kubectl describe no`, `kubectl get no -o yaml` 명령어를 사용해 no에 대한 상세 정보를 조회할 수 있다.

## Looking at logs
지금은 cluster에 대한 더 자세한 사항을 확인하기 위해서는 로그를 확인해야 한다. 관련 로그 파일 위치는 아래와 같다(systemd 기반 시스템에서는 journalctl을 대신 사용해야 할 수도 있다).

### Control Plane nodes
- `/var/log/kube-apiserver.log`: API Server, responsible for serving the API
- `/var/log/kube-scheduler.log`: Scheduler, responsible for making scheduling decisions
- `/var/log/kube-controller-manager.log`: a component that runs most Kubernetes built-in controllers, with the notable exception of scheduling (the kube-scheduler handles scheduling).

### Worker Nodes
- `/var/log/kubelet.log`: logs from the kubelet, responsible for running containers on the node
- `/var/log/kube-proxy.log`: logs from kube-proxy, which is responsible for directing traffic to Service endpoints

## Cluster failure modes
### Contributing causes
### Specific scenarios
### Mitigations