> 아래는 [metrics-server GitHub](https://github.com/kubernetes-sigs/metrics-server#requirements) 페이지를 번역한 내용이다

## Kubernetes Metrics Server
metrics server는 autoscaling 목적으로만 사용해야 한다. 예를 들어 이를 모니터링 솔루션을 위해 사용하면 안된다. 이러한 경우에는 직접 kubelet의 /metrics/reosource 엔드포인트에서 metric을 수집해야 한다.

### Use cases

### Requirements
metrics server는 클러스터, 네트워크 설정과 관련해 요구 사항이 있다. 이러한 요구 사항은 모든 클러스터 배포판에서 기본 값이 아닐 수 있다. metrics server를 사용하기 전에 먼저 클러스터 배포판에서 이러한 요구 사항을 지원하는지 확인한다:

- kube-apiserver는 aggregation layer를 활성화해야 한다.
- 
### Installation