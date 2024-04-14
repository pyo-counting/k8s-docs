aggregation layer는 core k8s API가 제공하는 기능 이외에 더 많은 기능을 제공할 수 있도록 추가 API를 통해 k8s를 확장할 수 있게 해준다. 추가 API는 [metrics server](https://github.com/kubernetes-sigs/metrics-server)와 같이 미리 만들어진 솔루션이거나 사용자가 직접 개발한 API일 수 있다.

aggregation layer는 kube-apiserver가 새로운 종류의 object 인식하도록 하는 방법인 crd와는 다르다. crd은 새로운 k8s resource를 추가하는 것이지만 aggregation layer는 kube-apiserver외 API를 추가하는 것이다.

## Aggregation layer
aggregation layer는 kube-apiserver 프로세스 안에서 동작한다. 확장 resource가 등록되기 전까지, aggregation layer는 아무 일도 하지 않는다. API를 등록하기 위해서 사용자는 k8s API 내에서 URL path를 "요구하는(claim)" APIService resource object를 추가해야 한다. 이때, aggregation layer는 해당 API path(예: /apis/myextensions.mycompany.io/v1/...)로 전송되는 모든 것을 등록된 APIService로 프록시하게 된다.

APIService를 구현하는 가장 일반적인 방법은 cluster 내에 실행되고 있는 po에서 extension API server를 실행하는 것이다. extension API server를 사용해서 cluster의 resource를 관리하는 경우 extension API server("extension-apiserver" 라고도 한다)는 일반적으로 하나 이상의 controller와 쌍을 이룬다. apiserver-builder library는 extension API server와 연관된 controller에 대한 골격을 제공한다.

### Response latency
extension-apiserver는 kube-apiserver로 오가는 연결에 대해 latency가 낮아야 한다. kube-apiserver의 discovery request은 왕복 latency가 5초 이내여야 한다.

extension API server가 latency에 대한 요구 사항을 달성할 수 없는 경우 이를 충족할 수 있도록 변경하는 것을 고려한다.