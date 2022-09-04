endpointslices resource는 ep resource과 비교해 확장 가능하다. endpointslices는 svc 뒤 po에 대한 IP 주소, port, readiness, topology에 대한 정보를 추적한다.

k8s 1.19에서 이 기능은 기본적으로 활성화된다. kube-proxy는 ep가 아닌 endpointslices를 이용한다. 이는 차이점이 없어보일 수 있지만 클러스터의 규모가 커지면 커질수록 확장성 향상이 있다. 또한 topology aware routing과 같은 k8s의 기능을 활성화한다.

### Scalability Limitations of the Endpoints API
기존 1개의 svc에 대해서는 오직 1개의 ep만 존재했다. 즉, 1개의 ep에 모든 po의 IP, port 정보를 저장해야 했다. 그 결과 막대한 API resource가 발생한다. kube-proxy는 모든 no에 실행되며 ep resource에 대한 업데이트를 watch한다. 작은 변경 사항에 있어 해당 ep object 전체를 각 kube-proxy에 전달되어야 한다.

뿐만 아니라 svc에 대해 관리될 수 있는 network endpoint의 수가 제한된다. etcd에 저장될 수 있는 객체의 기본 크기 제한은 1.5MB다. 이는 ep resource가 대략 5,000개의 po IP만 가질 수 있음을 의미한다. 이는 대부분의 사용자에게 있어 문제가되지 않지만 대규모 클러스터 환경에서는 심각한 문제가 된다.

### Splitting endpoints up with the EndpointSlice API
endpointslices는 샤딩(sharding)과 유사한 접근 방식으로 이 문제를 해결한다. svc에 대한 모든 po IP를 단일 ep resource에 관리하는 대신 여러 작은 endpointslices로 관리하는 것이다.

아래는 15개의 po에 대한 예시다.