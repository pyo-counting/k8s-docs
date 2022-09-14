> 아래는 [metrics-server GitHub](https://github.com/kubernetes-sigs/metrics-server#requirements) 페이지를 번역한 내용이다

## Kubernetes Metrics Server
metrics server는 autoscaling 목적으로만 사용해야 한다. 예를 들어 이를 모니터링 솔루션을 위해 사용하면 안된다. 이러한 경우에는 직접 kubelet의 /metrics/reosource 엔드포인트에서 metric을 수집해야 한다.

### Use cases

### Requirements
metrics server는 클러스터, 네트워크 설정과 관련해 요구 사항이 있다. 이러한 요구 사항은 모든 클러스터 배포판에서 기본 값이 아닐 수 있다. metrics server를 사용하기 전에 먼저 클러스터 배포판에서 이러한 요구 사항을 지원하는지 확인한다:

- kube-apiserver는 aggregation layer를 활성화돼야 한다.
- no의 kubelet에 대해서 authentication and authorization이 활성화돼야 한다.
- Kubelet certificate needs to be signed by cluster Certificate Authority (or disable certificate validation by passing --kubelet-insecure-tls to Metrics Server)
- Container runtime must implement a container metrics RPCs (or have cAdvisor support)
- Network should support following communication:
    - Control plane to Metrics Server. Control plane node needs to reach Metrics Server's pod IP and port 10250 (or node IP and custom port if hostNetwork is enabled). Read more about control plane to node communication.
    - Metrics Server to Kubelet on all nodes. Metrics server needs to reach node address and Kubelet port. Addresses and ports are configured in Kubelet and published as part of Node object. Addresses in .status.addresses and port in .status.daemonEndpoints.kubeletEndpoint.port field (default 10250). Metrics Server will pick first node address based on the list provided by kubelet-preferred-address-types command line flag (default InternalIP,ExternalIP,Hostname in manifests).

### Installation
metrics server는 YAML manifest 또는 helm chart를 통해 설치할 수 있다. 아래는 yaml manifect를 이용해 배포하는 방법이다:

``` bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```