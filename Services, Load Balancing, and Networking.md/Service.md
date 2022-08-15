## Service
k8s는 po에 고유한 IP를 할당한다. 그리고 svc를 이용하면 po 집합에 대한 단일 DNS 이름을 부여할 수 있으며 po 간에 로드 밸런싱을 할 수 있다.

### Motivation
k8s po는 클러스터에서 desired state를 만족하기 위해 생성되거나 삭제된다. po는 영속성을 갖는 reousrce가 아니다.

각 po는 각각의 IP를 할당받는다. 하지만 po는 영속성을 갖지 않으며 삭제 또는 생성될 수 있으며 이때마다 새로운 IP를 할당받을 것이다. 서비스를 제공하는 애플리케이션을 실행하는 po의 경우 IP가 바뀐다면 이를 이용하는 클라이언트 입장에서 계속해서 변경되는 po의 IP를 전달받아야 하는 문제가 있다. 이러한 문제를 해결하기 위해 svc를 사용할 수 있다.

### Service resources
k8s에서 svc는 po의 논리적 집합을 정의하며, 정의된 po 집합에 접근할 수 있는 정책을 정의하는 추상적 개념(때때로 이런 패턴을 micro-service라고 함)이다. service의 타겟이 되는 po의 집합은 selector에 의해 결정된다.

#### Cloud-native service discovery
If you're able to use Kubernetes APIs for service discovery in your application, you can query the API server for Endpoints, that get updated whenever the set of Pods in a Service changes.

For non-native applications, Kubernetes offers ways to place a network port or load balancer in between your application and the backend Pods.

