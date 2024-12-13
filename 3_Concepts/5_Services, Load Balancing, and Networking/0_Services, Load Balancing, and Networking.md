## The Kubernetes network model
- cluster의 모든 po는 고유한 cluster-wide IP 주소를 갖는다. 즉, po 간에 명시적으로 연결을 생성할 필요가 없으며 container 포트를 호스트 포트에 매핑하는 작업을 거의 수행할 필요가 없다. 이를 통해 포트 할당, 네이밍, service discovery, 로드 밸런싱, 애플리케이션 설정, 마이그레이션 관점에서 po를 VM 또는 물리적 호스트와 매우 유사하게 처리할 수 있는 모델을 구성한다.

k8s는 의도된 네트워크 분리 정책을 제외한 모든 네트워킹 구현에 대해 다음과 같은 기본 요구 사항을 갖는다:

- po는 NAT 없이 다른 no의 모든 po와 통신할 수 있어야 한다.
- no의 에이전트(예: system daemon, kubelet)가 해당 no의 모든 po와 통신할 수 있어야 한다.

*Note*: 호스트 네트워크에서 po가 실행되는 것을 지원하는 플랫폼(예: 리눅스)의 경우 po가 no의 호스트 네트워크에 연결되어 있어도 NAT 없이 모든 no의 모든 po와 통신할 수 있어야 한다.

이 모델은 덜 복잡할 뿐만 아니라 애플리케이션을 VM 환경에서 컨테이너로 마이그레이션하는데 공수가 적어야하는 k8s의 요구 사항에도 매칭된다. 이전 작업들이 VM에서 수행된 경우 각 VM은 IP가 있고 프로젝트의 다른 VM과 통신할 수 있다. 이는 동일한 기본적인 모델을 의믜한다.

k8s ip 주소는 po 범위(동일 po 내 container는 ip, MAC 주소를 포함하한 네트워크 네임스페이스를 공유)에 속한다. 즉, po 내의 container는 모두 로컬 호스트를 통해 서로의 포트에 접근할 수 있다. 또한 po 내의 container간에 포트 점유에 유의해야 함을 의미하지만, 이는 VM의 프로세스와 다르지 않다. 이를 "IP-per-pod" 모델이라고 한다.

이러한 내용이 구현되는 방법은 k8s에서 사용 중인 container runtime의 세부 구현 사항이다.

no에서 po로 포워딩하는 포트(host port)를 구현할 수 있지만, 이는 매우 틈새(niche) 작업이다. 포워딩 방법 역시 container runtime의 세부 구현 사항이다. po 자체는 호스트 포트의 존재 여부를 알지 못한다.

Kubernetes networking addresses four concerns:

- Containers within a Pod use networking to communicate via loopback.
- Cluster networking provides communication between different Pods.
- The Service API lets you expose an application running in Pods to be reachable from outside your cluster.
    - Ingress provides extra functionality specifically for exposing HTTP applications, websites and APIs.
- You can also use Services to publish services only for consumption inside your cluster.