k8s 구성요소는 prometheus format을 사용해 metric을 노출한다.

## Metrics in Kubernetes
대부분의 경우 metric은 HTTP 서버의 /metrics 엔드포인트에 노출된다. 기본적으로 엔드포인트를 노출하지 않는 구성요소의 경우 --bind-address 플래그를 사용하여 활성화할 수 있다.

구성요소의 예는 다음과 같다:

- kube-controller-manager
- kube-proxy
- kube-apiserver
- kube-scheduler
- kubelet

운영 환경에서는 이러한 metric을 주기적으로 수집하고 시계열 데이터베이스에서 사용할 수 있도록 prometheus 또는 다른 metric scraper를 구성할 수 있다.

참고로 kubelet도 /metrics/cadvisor, /metrics/resource, /metrics/probes 엔드포인트에서 metric을 노출한다. 이러한 metric은 동일한 라이프사이클을 가지지 않는다.

클러스터가 RBAC을 사용하는 경우 metric을 조회하기 위해 /metrics에 접근을 허용하는 클러스터롤(ClusterRole)을 가지는 사용자, 그룹 또는 sa를 통한 권한이 필요하다. 예를 들면, 다음과 같다:

``` yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus
rules:
  - nonResourceURLs:
      - "/metrics"
    verbs:
      - get
```

## Metric lifecycle