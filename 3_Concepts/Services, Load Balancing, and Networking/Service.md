k8s는 po에 고유한 IP를 할당한다. 그리고 svc를 이용하면 po 집합에 대한 단일 DNS 이름을 부여할 수 있으며 po 간에 로드 밸런싱을 할 수 있다.

## Motivation
k8s po는 클러스터에서 desired state를 만족하기 위해 생성되거나 삭제된다. po는 영속성을 갖는 reousrce가 아니다.

각 po는 각각의 IP를 할당받는다. 하지만 po는 영속성을 갖지 않으며 삭제 또는 생성될 수 있으며 이때마다 새로운 IP를 할당받을 것이다. 서비스를 제공하는 애플리케이션을 실행하는 po의 경우 IP가 바뀐다면 이를 이용하는 클라이언트 입장에서 계속해서 변경되는 po의 IP를 전달받아야 하는 문제가 있다. 이러한 문제를 해결하기 위해 svc를 사용할 수 있다.

## Service resources
k8s에서 svc는 po의 논리적 집합을 정의하며, 정의된 po 집합에 접근할 수 있는 정책을 정의하는 추상적 개념(때때로 이런 패턴을 micro-service라고 함)이다. service의 타겟이 되는 po의 집합은 `.spec.selector`에 의해 결정된다.

### Cloud-native service discovery
If you're able to use Kubernetes APIs for service discovery in your application, you can query the API server for Endpoints, that get updated whenever the set of Pods in a Service changes.

For non-native applications, Kubernetes offers ways to place a network port or load balancer in between your application and the backend Pods.

## Defining a Service
k8s는 svc proxy에서 사용되는 IP 주소(cluster IP라고도 부름)를 svc에 할당한다.

svc controller는 .spec.selector에 매칭되는 po를 계속해서 찾으며 svc와 동일한 이름의 endpoint resource object를 업데이트한다.

**Note**: 일반적으로 `port`와 `targetPort`를 갖은 값으로 설정한다.

po 정의 시 port에 이름을 정의할 수 있으며 이 이름은 svc의 `targetPort`에서 사용할 수 있다.

svc의 기본 프로토콜은 TCP다.

### Services without selectors
svc는 일반적으로 .spec.selector 필드를 사용해 k8s 내 po에 대한 접근을 추상화하지만 selector 대신 직접 ep object를 구성함으로써 다른 종류의 백엔드도 추상화할 수 있으며 심지어 클러스터 외부에서 실행되는 것들도 포함할 수 있다.

k8s 클러스터의 api server의 경우 호스트 네트워크를 사용하는 po로 구성된다. 그리고 해당 po는 selector가 없는 svc로 구성된다.

svc 내 `spec.selecotr`를 명시하지 않으면 관련된 ep object는 자동으로 생성되지 않는다. 그렇기 때문에 사용자는 직접 ep object를 생성해야 한다.

ep의 이름은 svc와 동일해야 한다.

**Note**: ep object의 IP는 loopback(127.0.0.0/8), link-local(169.254.0.0/16)이면 안된다.

ep IP는 k8s svc의 cluster IP이면 안된다. 왜냐하면 kube-proxy는 virtual IP를 destination으로 지원하지 않기 떄문이다.

`.spec.type`이 ExternalName에 대해서 `.spec.selector`가 필요없으며 대신 `.spec.externalName`이 필요히다.

### Over Capacity Endpoints
endpoint resource에 1,000개가 넘는 IP가 설정될 경우 쿠버네티스 v1.22이상에서 클러스터는 endpoint resource에 endpoints.kubernetes.io/over-capacity: truncated annotation을 추가한다. 이 annotation은 해당 endpoint object가 용량을 초과했으며 endpoint controller가 엔드포인트의 수를 1000으로 줄였음(truncate)을 나타낸다.

### EndpointSlices
endpointslices는 ep에 비해 더 확장성있는 대안을 제공한다. ep와 개념적으로 유사하지만 endpointslices는 여러 resource에 네트워크 엔드포인트를 분산시킬 수 있다. 기본적으로 endpointslices는 최대 100개의 엔드포인트를 가질 수 있으며 추가 엔드포인트를 저장하기 위해서는 추가 endpointslices가 생성된다.

### Application protocol