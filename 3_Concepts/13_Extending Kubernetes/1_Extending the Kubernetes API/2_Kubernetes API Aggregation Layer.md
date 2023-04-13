aggregation layer는 core k8s API가 제공하는 기능 이외에 더 많은 기능을 제공할 수 있도록 추가 API를 통해 k8s를 확장할 수 있게 해준다. 추가 API는 metrics server와 같이 미리 만들어진 솔루션이거나 사용자가 직접 개발한 API일 수 있다.

aggregation layer는 kube-apiserver가 새로운 종류의 object 인식하도록 하는 방법인 Custom Resources와는 다르다.

## Aggregation layer
aggregation layer는 kube-apiserver 프로세스 안에서 동작한다. 확장 resource가 등록되기 전까지, aggregation layer는 아무 일도 하지 않는다. API를 등록하기 위해서 사용자는 k8s API 내에서 URL 경로를 "요구하는(claim)" APIService resource object를 추가해야 한다. 이때, aggregation layer는 해당 API 경로(예: /apis/myextensions.mycompany.io/v1/...)로 전송되는 모든 것을 등록된 APIService로 프록시하게 된다.

APIService를 구현하는 가장 일반적인 방법은 클러스터 내에 실행되고 있는 po에서 extension API server를 실행하는 것이다. extension API server를 사용해서 클러스터의 resource를 관리하는 경우 extension API server("extension-apiserver" 라고도 한다)는 일반적으로 하나 이상의 controller와 쌍을 이룬다. apiserver-builder 라이브러리는 extension API server와 연관된 controller에 대한 골격을 제공한다.

### Response latency
extension-apiserver는 kube-apiserver로 오가는 연결에 대해 지연이 낮아야 한다. kube-apiserver로 부터의 디스커버리 요청은 왕복 지연이 5초 이내여야 한다.

extension API server가 지연에 대한 요구 사항을 달성할 수 없는 경우 이를 충족할 수 있도록 변경하는 것을 고려한다.


