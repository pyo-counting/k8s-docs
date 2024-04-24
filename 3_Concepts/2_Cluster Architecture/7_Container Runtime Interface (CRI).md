CRI(Container Runtime Interface)는 kubelet이 cluster 구성 요소를 다시 컴파일할 필요 없이 다양한 container runtime을 사용 가능하게 해주는 plugin interface다.

cluster의 각 no는 kubelet이 po, container를 실행할 수 있도록 container runtime이 필요하다.

container runtime interface(CRI)는 kubelet과 container runtime 사이의 프로토콜이다.

k8s container runtime interface(CRI)는 no의 구성 요소인 kubelet과 container runtime 사이의 gRPC 프로토콜을 정의한다.

## The API
kubelet은 gRPC를 사용해 container에 연결할 때 client로 동작한다. runtime endpoint, image service endpoint는 container runtime에서 사용 가능해야 하며 kubelet에서는 `.containerRuntimeEndpoint`, `.imageServiceEndpoint` 설정을 사용해 container runtime의 endpoint를 명시해야 한다.

k8s v1.29에서는 kubelet이 CRI v1을 사용하는 것을 선호한다. container runtime이 CRI의 v1을 지원하지 않는 경우 kubelet은 이전 버전의 지원되는 버전으로 협상하려고 시도한다. v1.29 kubelet은 CRI v1alpha2를 협상할 수 있지만 이 버전은 deprecated 된 것으로 간주된다. kubelet과 container runtime 간의 협상이 실패할 경우 kubelet은 포기하고 no를 등록하지 않는다.

## Upgrading
k8s를 업그레이드 시, 구성 요소가 재시작할 때 kubelet은 최신 CRI를 자동으로 선택하려고 시도한다. 하지만 이 작업이 실패할 경우 위에서 언급한 절차를 따른다. If a gRPC re-dial was required because the container runtime has been upgraded, then the container runtime must also support the initially selected version or the redial is expected to fail. This requires a restart of the kubelet.

