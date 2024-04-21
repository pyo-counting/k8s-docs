k8s API의 k8s object state는 metric으로 노출할 수 있다. [kube-state-metrics](https://github.com/kubernetes/kube-state-metrics)은 kube-apiserver로부터 cluster의 각 object state 기반으로 생성한 metric을 HTTP endpoint로 노출한다. label, annotation, startup과 termination time, status, object의 현재 phase와 같은 다양한 state 정보를 노출한다. 예를 들어 po에서 실행되는 container는 `kube_pod_container_info` metric을 생성한다. 해당 metric에는 container 이름, container가 속한 po의 이름, po가 실행중인 ns, container image 이름과 id등과 같은 정보를 포함한다.

> This item links to a third party project or product that is not part of Kubernetes itself. More information

kube-state-metrics의 endpoint를 scrape할 수 있는 구성 요소를 사용해 아래와 같은 예시에 활용할 수 있다.

## Example: using metrics from kube-state-metrics to query the cluster state 
kube-state-metrics이 생성한 metric은 쿼리에 사용할 수 있기 때문에 cluster에 대한 추가 통찰력을 얻을 수 있다.

아래 prometheus의 PromQL 쿼리는 not ready po의 수를 반환한다.
```
count(kube_pod_status_ready{condition="false"}) by (namespace, pod)
```

## Example: alerting based on from kube-state-metrics
kube-state-metrics이 생성한 metric을 사용해 alert을 구성할 수 있다.

아래 prometheus의 alert rule language를 사용해 5분 이상 `Terminating` 상태의 po가 있을 경우 알람을 발생시킨다.
```
groups:
- name: Pod state
  rules:
  - alert: PodsBlockedInTerminatingState
    expr: count(kube_pod_deletion_timestamp) by (namespace, pod) * count(kube_pod_status_reason{reason="NodeLost"} == 0) by (namespace, pod) > 0
    for: 5m
    labels:
      severity: page
    annotations:
      summary: Pod {{$labels.namespace}}/{{$labels.pod}} blocked in Terminating state.
```