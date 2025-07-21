## The Kubernetes network model
k8s 네트워크 모델은 여러 부분으로 구성된다.
- cluster의 모든 po는 고유한 cluster-wide IP 주소를 갖는다.
    - po는 자체적인 private network namespace를 갖는다(동일 po 내 container가 공유). 동일한 po 내에서 다른 container에서 실행되는 프로세스도 localhost를 통해 서로 통신할 수 있다.
- po network(또는 cluster network)는 po 간의 통신을 처리한다. 의도적인 네트워크 분할이 없는 한 다음을 보장한다.
    - 모든 po는 동일한 no에 있든 다른 no에 있든 상관없이 다른 모든 po와 통신할 수 있다. po는 프록시나 NAT 없이 직접 서로 통신할 수 있다(On Windows, this rule does not apply to host-network pods).
    - no의 agent(시스템 데몬, kubelet 등)는 해당 no의 모든 po와 통신할 수 있다.
- svc를 사용하면 하나 이상의 백엔드 po에 의해 구현되는 서비스에 대해 안정적인 IP 주소나 hostname을 제공할 수 있다. 이때 svc를 구성하는 각각의 po는 시간이 지남에 따라 변경될 수 있다.
    - k8s는 현재 svc를 지원하는 po에 대한 정보를 제공하기 위해 endpointslices를 자동으로 관리한다.
    - service proxy 구현체는 svc, endpointslices를 모니터링하고 OS나 cloud provider API를 사용해 패킷을 가로채거나 재작성함으로써 서비스 트래픽을 백엔드로 라우팅하는 data plane을 구성한다.
- gateway(또는 이전 버전인 ing)를 사용해 cluster 외부의 클라이언트가 서비스에 접근할 수 있도록 할 수 있다.
    - 일부 cloud provider의 경우 svc의 `type: LoadBalancer`를 통해 더 간단하지만 설정 기능이 적은 ing를 사용할 수 있다.
- np는 po 간 또는 po와 외부 세계 간의 트래픽을 제어할 수 있게 한다.

구형 container 시스템에서는 다른 host에 있는 container에 자동 연결 기능이 없었다. 따라서 container 간에 명시적으로 링크를 생성하거나, 다른 host의 container가 접근할 수 있도록 container port를 host port에 매핑해야 하는 경우가 많았다. k8s에서는 이런 작업이 필요 없다. k8s의 모델은 port 할당, 이름 지정, service discovery, 로드 밸런싱, 애플리케이션 설정, 마이그레이션 관점에서 po를 VM(가상 머신)이나 물리적 host처럼 취급할 수 있다는 것이다.

이 모델 중 일부만 k8s 자체에서 구현된다. 나머지 부분에 대해서는 k8s가 API를 정의하지만, 해당 기능은 외부 구성요소에 의해 제공되며 그중 일부는 선택 사항이다.
- po network namespace 설정은 cri를 구현하는 system-level 소프트웨어서 다룬다.
- po network 자체는 po network 구현체에 의해 관리된다. 리눅스에서 대부분의 container runtime은 cni를 사용해 [po network 구현체](https://kubernetes.io/docs/concepts/cluster-administration/addons/#networking-and-network-policy)와 상호 작용하므로, 이러한 구현체들은 CNI plugin이라고 불린다.
- k8s는 kube-proxy라는 기본 서비스 프록시 구현체를 제공하지만 일부 po network 구현체는 나머지 구현과 더 긴밀하게 통합된 자체 서비스 프록시를 대신 사용한다.
- np 역시 일반적으로 po network 구현체에 의해 구현된다. (일부 po network 구현체는 np를 구현하지 않거나, 관리자가 np 지원 없이 po network를 구성하도록 선택할 수 있다. 이런 경우 API는 존재하지만 아무런 효과가 없다)
- [gateway 구현체](https://gateway-api.sigs.k8s.io/implementations/)들이 많이 있으며 일부는 특정 cloud 환경에 특화되어 있고 일부는 bare metal 환경에 더 중점을 둔다.

