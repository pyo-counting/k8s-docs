- .spec.selctor는 .spec.type이 ClusterIP, NodePort, LoadBalancer일 떄만 적용됨. ExternalName일 때는 무시된다고 documentation에 명시되어 있지만, ep는 생성됨을 확인(물론 프록시는 되지 않음). 
- .spec.selector가 없는 svc는 일반적인 svc와 동일하게 동작한다. 하지만 ep를 자동으로 생성하지 않으며 사용자가 직접 생성/관리해야 하는 책임이 있다.
- .spec.type을 ExternalName으로 바꾸거나 반대로 바꾸지 않는 이상 svc가 수정되더라도 clusterIP는 고정 값이다.
- .spec.clusterIP는 .spec.type이 NodePort, ClusterIP, LoadBalancer일 때 자동 할당되거나 사용자가 설정할 수 있다.
- .spec.clusterIP가 None: headless svc를 의미(cluster ip 미할당). kube-proxy가 해당 svc를 다루지 않기 때문에 프록시, 로드밸런싱되지 않는다. 즉 .spec.ports[*]를 명시해도 의미가 없다.
  - .spec.selector가 있으면 svc DNS lookup 시 pod들의 ip 목록(A 레코드)가 조회됨. ep도 생성됨. .spec.clusterIP: "None"이 아니면 오류. 조회된 pod의 ip와 po의 노출 po로 접근할 수 있다.
  - .spec.selector가 없으면 svc DNS lookup 시 아무것도 조회 안됨. 무슨 용도인지 모르겠음
- .spec.type이 ClusterIP: \<cluster ip>:\<port>로 접근 가능
- .spec.type이 NodePort: \<node ip>: \<node port>, \<cluter ip>:\<port>로 접근 가능 
- .spec.type이 ExternalName: .spec.externalName 필드 필수로 설정 필요. .spec.clusterIP 핃드를 명시하지 않거나 값을 ""으로 설정 필요. cluster ip, port 할당되지 않음. svc DNS lookup 시 CNAME 레코드 조회됨.
- .spec.externalIPs[*]는 .spec.type이 어떤 값이여도 같이 사용할 수 있다. 


k8s는 po에 고유한 ip를 할당한다. 그리고 svc를 이용하면 po 집합에 대한 단일 DNS 이름을 부여할 수 있으며 po 간에 로드 밸런싱을 할 수 있다.

## Motivation
k8s po는 클러스터에서 desired state를 만족하기 위해 생성되거나 삭제된다. po는 영속성을 갖는 reousrce가 아니다. deploy를 사용해 애플리케이션을 실행하면, deploy가 동적으로 po를 생성 및 삭제한다.

각 po는 각각의 ip를 할당받는다. 하지만 po는 영속성을 갖지 않으며 삭제 또는 생성될 수 있으며 이때마다 새로운 ip를 할당받는다. 서비스를 제공하는 애플리케이션을 실행하는 po의 경우 ip가 바뀐다면 이를 이용하는 클라이언트 입장에서 계속해서 변경되는 po의 ip를 전달받아야 하는 문제가 있다. 이러한 문제를 해결하기 위해 svc를 사용할 수 있다.

## Service resources
k8s에서 svc는 po의 논리적 집합을 정의하며, 정의된 po 집합에 접근할 수 있는 정책을 정의하는 추상적 개념(때때로 이런 패턴을 micro-service라고 함)이다. svc의 타겟이 되는 po의 집합은 `.spec.selector`에 의해 결정된다. svc의 endpoint를 정의하는 다른 방법은 Services without selectors 페이지를 참고한다.

### Cloud-native service discovery
애플리케이션에서 service discovery를 위한 k8s API를 이용할 수 있다면 svc의 po 집합이 변경될 때마다 업데이트가 되는 ep를 위한 쿼리를 수행할 수 있다.

이용하지 못할 경우 k8s의 network port 또는 load balancer를 이용할 수 있다.

## Defining a Service
svc는 cm, po와 같은 오브젝트로 정의할 수 있으며 kubectl 명령어를 사용해 조회할 수 있다.

예를 들어 TCP 9376 포트에서 listen하며 app.kubernetes.io/name=MyApp label을 갖는 po의 집합이 있을 때, po들을 pubkush하기 위해 svc를 정의할 수 있다:

``` yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app.kubernetes.io/name: MyApp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
```

k8는 svc에 ip 주소(cluster ip)를 할당한다. 해당 ip는 virtual ip 주소 메커니즘을에 이용된다. 관련 내용은 Virtual IPs and service proxies 페이지를 참고한다.

svc controller는 .spec.selector에 매칭되는 po를 계속해서 찾으며 svc와 동일한 이름의 ep resource object를 업데이트한다.

**Note**: 일반적으로 `port`와 `targetPort`를 갖은 값으로 설정한다.

po 정의 시 port에 이름을 정의할 수 있으며 이 이름은 svc의 `targetPort`에서 사용할 수 있다. 아래는 예시다:

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app.kubernetes.io/name: proxy
spec:
  containers:
  - name: nginx
    image: nginx:stable
    ports:
      - containerPort: 80
        name: http-web-svc

---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app.kubernetes.io/name: proxy
  ports:
  - name: name-of-service-port
    protocol: TCP
    port: 80
    targetPort: http-web-svc
```

이를 통해 po의 포트 번호를 유연하게 변경할 수 있다. svc의 기본 프로토콜은 TCP다.

svc에 여러 포트를 노출하는 것도 가능하다. 각 포트는 다른 프로토콜을 이용해도 괜찮다.

### Services without selectors
svc는 일반적으로 .spec.selector 필드를 사용해 k8s 내 po에 대한 접근을 추상화하지만 selector 대신 직접 ep object를 구성함으로써 다른 종류의 백엔드도 추상화할 수 있으며 심지어 클러스터 외부에서 실행되는 것들도 포함할 수 있다. 예를 들어:

- You want to have an external database cluster in production, but in your test environment you use your own databases.
- svc가 다른 ns 또는 클러스터의 svc를 가리키도록 하기 위한 경우
- You are migrating a workload to Kubernetes. While evaluating the approach, you run only a portion of your backends in Kubernetes.

.spec.selector 없이 svc를 정의한다. 아래는 예시다:

``` yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
```

svc에 .spec.selector가 없기 때문에 관련된 ep는 자동으로 생성되지 않는다. 대신 직접 ep를 생성해 svc와 매핑할 수 있다.

``` yaml
apiVersion: v1
kind: Endpoints
metadata:
  # the name here should match the name of the Service
  name: my-service
subsets:
  - addresses:
      - ip: 192.0.2.42
    ports:
      - port: 9376
```

ep의 이름은 svc와 동일해야 한다.

k8s 클러스터의 api server의 경우 호스트 네트워크를 사용하는 po로 구성된다. 그리고 해당 po는 selector가 없는 svc로 구성된다.

**Note**: ep object의 ip는 loopback(127.0.0.0/8 for IPv4, ::1/128 for IPv6), link-local((169.254.0.0/16 and 224.0.0.0/24 for IPv4, fe80::/64 for IPv6)이면 안된다.

ep ip는 k8s svc의 cluster ip이면 안된다. 왜냐하면 kube-proxy는 virtual ip를 destination으로 지원하지 않기 떄문이다.

selector가 없는 svc도 동일하게 동작한다. 위 예시의 경우 svc로 인입되는 트래픽은 ep에 정의된 192.0.2.42:9376(TCP)로 라우팅된다.

**Note**: The Kubernetes API server does not allow proxying to endpoints that are not mapped to pods. Actions such as kubectl proxy \<service-name\> where the service has no selector will fail due to this constraint. This prevents the Kubernetes API server from being used as a proxy to endpoints the caller may not be authorized to access.

`.spec.type`이 ExternalName인 svc의 경우 `.spec.selector`가 필요없으며 대신 `.spec.externalName`이 필요히다.

### Over Capacity Endpoints
ep resource에 1,000개가 넘는 ip가 설정될 경우 쿠버네티스 v1.22이상에서 클러스터는 ep resource에 endpoints.kubernetes.io/over-capacity: truncated annotation을 추가한다. 이 annotation은 해당 ep object가 용량을 초과했으며 endpoint controller가 엔드포인트의 수를 1000으로 줄였음(truncate)을 나타낸다.

### EndpointSlices
endpointslices는 ep에 비해 더 확장성있는 대안을 제공한다. ep와 개념적으로 유사하지만 endpointslices는 여러 resource에 네트워크 엔드포인트를 분산시킬 수 있다. 기본적으로 endpointslices는 최대 100개의 엔드포인트를 가질 수 있으며 추가 엔드포인트를 저장하기 위해서 추가 endpointslices가 생성된다.

### Application protocol
.spec.ports[*].appProtocol 필드는 각 svc 포트에 대한 애플리케이션 프로토콜을 명시하는 방법이다. 해당 필드의 값은 ep, endpointslices 객체에서 미러링한다.

해당 필드는 표준 k8s label 문법을 따른다. 값은 [IANA standard service names](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml) 또는 mycompany.com/my-custom-protocol와 같이 사용할 수 있다.

## Virtual IPs and service proxies
k8s 클러스터의 모든 no에는 kube-proxy가 실행된다. kube-proxy는 `.spec.type`필드가 ExternalName가 아닌 svc의 가상 ip 형식을 구현하는 역할을 담당한다.

### Why not use round-robin DNS?

**Note**: [TTL(Time to Live)](http://www.codns.com/b/B05-115)은 DNS 레코드의 변경사항이 적용될 때까지 걸리는 시간(초)을 결정하는 DNS 레코드 값으로 A 레코드,MX 레코드, CNAME 레코드 등 도메인의 각 DNS 레코드에는 TTL 값이 함께 적용되어 사용하게 된다. TTL은 타 네임서버가 해당 도메인의 정보(예, www.codns.com=1.234.37.233)를 메모리상에 임시 저장하는 시간(캐싱, 도메인 정보의 생존 시간)으로 프로그램이나 브라우저에서 도메인을 입력할 경우, 먼저 컴퓨터의 임시 저장된 주소 정보에 묻고, 만약 저장정보가 없으면, 등록해놓은 DNS(보통 해당 통신사의 DNS)를 통하여 주소정보를 받아 ipP로 연결하게 되는 것

k8s가 인바운드 트래픽을 백엔드로 전달하기 위해 프록시를 사용하는 이유는 무엇일까? svc를 위해 프로시를 사용하는 몇 가지 이유는 다음과 같다:

- DNS 구현 시 레코드 TTL을 무시함으로써 만료된 이후에도 캐싱을 유지하는 경우도 있다.
- 일부 애플리케이션의 경우 DNS lookup을 한 번만 수행하고 결과를 무한정 캐시한다.
- 애플리케이션과 라이브러리가 다시 DNS lookup을 수행하더라도 DNS 레코드의 TTL이 낮거나 0이면 DNS에 높은 부하가 걸려 관리가 어려워질 수 있다.

해당 주제에서는 다양한 kube-proxy 구현에 대해 설명한다. 공통적으로 kube-proxy가 실행될 때 커널 레벨의 규칙이 수정될 수 있다(예: iptables 규칙이 생성될 수 있음). 어떤 경우에는 재부팅할 때까지 삭제가 되지 않을 수도 있다. 따라서 kube-proxy를 실행하는 것은 컴퓨터에서 low lavel의 권한을 가진 네트워크 프록시 서비스의 결과를 이해하는 관리자만이 수행해야 하는 작업이다. kube-proxy 실행 파일은 cleanup 기능을 지원하지만 이 기능은 공식 기능이 아니므로 as-is로만 사용할 수 있다.

### Configuration
kube-proxy는 설정에 따라 다양한 모드로 실행될 수 있다.

- The kube-proxy's configuration is done via a ConfigMap, and the ConfigMap for kube-proxy effectively deprecates the behaviour for almost all of the flags for the kube-proxy.
- The ConfigMap for the kube-proxy does not support live reloading of configuration.
- The ConfigMap parameters for the kube-proxy cannot all be validated and verified on startup. For example, if your operating system doesn't allow you to run iptables commands, the standard kernel kube-proxy implementation will not work. Likewise, if you have an operating system which doesn't support netsh, it will not run in Windows userspace mode.

### User space proxy mode
이 legacy 모드에서 kube-proxy는 k8s control plane을 watch하며 svc, ep 객체의 추가/삭제를 확인한다. 각 svc에 대해 로컬 no에 임의의 포트를 오픈한다. 해당 "proxy port"에 대한 모든 연결은 svc의 po 중 하나로 프록시된다(ep의 목록을 svc에 연결된 po를 확인). kube-proxy는 여러 po 중 .spec.SessionAffinity 설정을 고려해 선택한다.

마지막으로 user-space mode의 kube-proxy는 svc의 cluster ip와 port로 인입되는 트래픽을 가로채기 위한 iptables 규칙을 설치한다. 해당 규칙은 트래픽을 proxy port로 리다이렉트한다.

기본적응로 use-space mode의 kube-proxy는 round-robin 알고리즘을 사용해 연결할 po를 선택한다.

> client 요청(cluster ip) -> iptables -> kube-proxy의 proxy port -> po

![user space mode](https://v1-24.docs.kubernetes.io/images/docs/services-userspace-overview.svg)

### iptables proxy mode

### IPVS proxy mode

## Multi-Port Services
일부 svc의 경우 여러 포트를 노출해야할 수도 있다. svc에 여러 포트를 사용할 경우 용도를 명확하게 하기 위해 이름을 명시해야한다. 아래는 예시다:

``` yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app.kubernetes.io/name: MyApp
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 9376
    - name: https
      protocol: TCP
      port: 443
      targetPort: 9377
```

**Note**: As with Kubernetes names in general, names for ports must only contain lowercase alphanumeric characters and -. Port names must also start and end with an alphanumeric character.

For example, the names 123-abc and web are valid, but 123_abc and -web are not.

## Choosing your own IP address
svc 생성 요청 시, cluster ip를 직접 .spec.clusterIP 필드에 명시할 수도 있다. 예를 들어 재사용하려는 기존 DNS 엔트리가 있거나 특정 ip 주소에 대해 재설정이 어려운 레거시 시스템이 있는 경우에 사용할 수 있다.

이 때 ip 주소는 API server에 설정된 service-cluster-ip-range CIDR 범위 내의 유효한 IPv4 또는 IPv6 주소여야 한다. 잘못된 clusterIP 값을 이용해 svc를 생성 요청할 경우 API server는 문제가 있음을 나타내는 422 HTTP status code를 반환한다.

## Traffic policies
### External traffic policy
.spec.externalTrafficPolicy 필드를 사용해 외부(externally-facing 주소: NodePorts, ExternalIPs, LoadBalancer IP)에서 인입되는 트래픽을 제어할 수 있다. 해당 필드에 사용 가능한 값은 Cluster, Local이 있다:

- Cluster: 기본값. 모든 ep로 균일하게 라우팅을 수행한다.
- Local: kube-proxy는 외부 load balacner가 no 간 트래픽 균형을 제어한다고 가정한다. 그렇기 때문에 각 no는 트래픽을 해당 no의 ep로만 전달한다(no에 ep가 없다면 트래픽은 버려진다).

### Internal traffic policy
.spec.internalTrafficPolicy 필드를 사용해 내부(.spec.clusterIP)에서 인입되는 트래픽을 제어할 수 있다. 해당 필드에 사용 가능한 값은 Cluster, Local이 있다:

- Cluster: 기본값. 클러스터에 존재하는 모든 ep 내 po로 라우팅
- Local: 요청된 po와 동일한 no에 존재하는 ep 내 po로 라우팅한다. 만약 동일 no에 엔드포인트 po가 없을 경우 트래픽은 버려진다.

## Discovering services
k8s는 svc를 찾기위한 2가지 방법을 제공한다: 환경 변수와 DNS

### Environment varibles
po가 no에 실행될 때 kubelet은 각 svc의 환경 변수 집합을 po에 추가한다. 환경 변수 이름은 `{SVCNAME}_SERVICE_HOST and {SVCNAME}_SERVICE_PORT` 형태를 갖으며, svc 이름은 대문자와 언더바(dash는 언더바로 치환)로 구성된다. It also supports variables (see makeLinkVariables) that are compatible with Docker Engine's "legacy container links" feature.

아래는 redis-master 이름, 6379/TCP LISTEN 포트, cluster ip가 10.0.0.11인 svc에 대한 환경 변수 목록이다.

``` bash
REDIS_MASTER_SERVICE_HOST=10.0.0.11
REDIS_MASTER_SERVICE_PORT=6379
REDIS_MASTER_PORT=tcp://10.0.0.11:6379
REDIS_MASTER_PORT_6379_TCP=tcp://10.0.0.11:6379
REDIS_MASTER_PORT_6379_TCP_PROTO=tcp
REDIS_MASTER_PORT_6379_TCP_PORT=6379
REDIS_MASTER_PORT_6379_TCP_ADDR=10.0.0.11
```

**Note**: svc에 접근하기 위한 po가 있을 경우, po가 생성되기 전에 먼저 svc가 생성되어야 한다. 그렇지 않으면 svc 관련 환경 변수가 설정되지 않는다.

물론 DNS를 사용하면 위와 같은 순서에 대한 문제는 고려하지 않아도 된다.

### DNS
k8s add-on을 사용해 k8s 클러스터를 위한 DNS를 설정할 수 있다.

CoreDNS와 같은 클러스터 레벨의 DNS 서버는 새롭게 생성되는 svc와 이에 대한 DNS 레코드를 생성하기 위해 k8s API를 watch한다. 클러스터 내 DNS가 활성화되면 모든 po는 svc를 DNS 이름을 통해 resolve 가능하다.

예를 들어 my-ns ns에 my-service svc가 존재할 경우, control plane과 DNS 서비스는 my-service.my-ns라는 DNS 레코드를 생성한다. my-ns ns 내 po는 ns를 생략한 my-service 도메인을 통해 lookup이 가능하다.

다른 ns의 po는 my-service.my-ns라는 이름으로 lookup이 가능하다. 해당 도메인은 svc에 할당된 cluster ip로 resolve 된다.

또한 k8s는 이름이 지정된 포트에 대한 DNS SRV(SeRVice) 레코드를 지원한다. 만약 my-service.my-ns svc가 http 이름을 갖으며 TCP 프로토콜을 사용하는 포트가 있을 경우, _http._tcp.my-service.my-ns DNS SRV 쿼리를 통해 http에 대한 포트 번호와 ip 주소를 lookup할 수 있다.

k8s DNS 서버는 ExternalName svc에 접근할 수 있는 유일한 방법이다. 자세한 내용은 DNS Pods and Services 페이지를 참고한다.

## Headless Services
로드 밸런싱 기눙과 단일 서비스 ip가 필요하지 않을 수도 있다. 이 경우 .spec.clusterIP 필드에 None 값을 지정해 "headless" svc를 생성할 수 있다.

headless svc를 사용해 k8s 구현에 얽메이지 않으면서 service discovery 메커니즘을 사용할 수 있다.

headless svc를 위해 cluster ip가 할당되지 않아야한다. 즉, kube-proxy가 해당 svc를 다루지 않는다. 그렇기 때문에 로드 밸런싱, 프록시 기능을 지원하지 않는다. DNS의 설정은 아래 selector의 정의 여부에 따라 다르다:

### With selectors
selector가 있는 headless svc는 endpoint controller가 ep를 생성한다. 대신 A 레코드(po의 ip주소들)를 반환하도록 DNS 설정을 수정한다.

### Without selectors
selecotr가 없는 headless svc는 endpoint controller가 ep를 생성하지 않는다. 대신 DNS 시스템은 아래 내용을 찾아 설정한다:

- ExternalName 타입 svc에서 사용할 CNAME 레코드
- 다른 모든 타입에 대해 svc와 이름을 공유하는 ep에 대한 레코드

## Publishing Services (ServiceTypes)
svc의 .spec.type을 사용해 여러 타입의 svc를 생성할 수 있다. 기본 값은 ClusterIP다.

- ClusterIP: (기본값) 클러스터 내부 ip를 통해 svc를 노출한다. svc는 클러스터 내부에서만 접근할 수 있다.
- NodePort: 각 no의 ip / static 포트(NodePort)에 svc를 노출한다. NodePort 타입의 svc는 라우팅을 위해 ClusterIP를 자동으로 생성한다. \<NodeIP>:\<NodePort>를 통해 클러스터 밖에서도 NodePort svc에 접근할 수 있다.
- LoadBalancer: cloud provider의 load balancer를 사용해 외부에 svc를 노출한다. 외부 load balancer가 라우팅을 위해 NodePort, ClusterIP를 자동으로 생성한다.
- ExternalName: 값이 포함된 DNS CNAME 레코드를 반환해 svc를 externalname 필드의 값(예를 들어 foo.bar.example.com)에 매핑한다. 어떤 종류의 프록시도 설정되지 않는다.

**Note**: ExternalName 타입을 사용하기 위해 kube-dns 1.7 버전 또는 CoreDNS 0.0.8 버전 이상을 사용해야 한다.

svc를 노출하기 위해 ingress를 사용할 수도 있다. ingress는 svc의 종류가 아니지만 클러스터의 진입점(entry point)역할을 수행한다.

### Type NodePort
k8s control plane은 --service-node-port-range flag(기본값: 30000-32767)에 명시된 범위에서 랜덤 포트를 할당한다. 각 no는 해당 프토를 svc에 프록시한다. svc는 .spec.ports[*].nodePort 필드에 할당된 포트를 설정한다.

포트를 프록시할 특정 ip를 지정하기 위해 kube-proxy에 --nodeport-addresses flag를 설정하거나 설정 파일에 nodePortAddresses 필드를 설정할 수 있다.

이 flag를 쉼표로 구분된 ip 목록이며 kube-proxy가 해당 no의 로컬로 간주한다.

예를 들어, --nodeport-addresses=127.0.0.0/8 flag를 지정하면 kube-proxy는 NodePort svc를 위해 루프백 인터페이스만 선택한다. --nodeport-addresses의 기본값은 빈 값이다. 즉, kube-proxy는 NodePort를 위해 이용 가능한 모든 네트워크 인터페이스를 고려해야 한다.

no의 포트를 지정하기 위해 svc의 .spec.ports[*].nodePort 필드를 사용할 수 있다. conrol plane은 해당 포트를 할당하거나 API 요청에 대한 실패를 반환한다. 사용자는 포트 충돌을 고려해야 한다는 것이다. 뿐만 아니라 NodePort 사용을 위해 설정된 범위 내의 값을 사용해야 한다.

NodePort를 사용하면 k8s에서 지원하지 않는 환경을 위한 로드 밸런싱 솔루션을 설정할 수 있다. 그리고 하나 이상의 no의 ip를 직접 노출할 수 있다.

NodePort svc는 \<NodeIP>:spec.ports[*].nodePort, spec.clusterIP:spec.ports[*].port를 통해 접근할 수 있다. If the --nodeport-addresses flag for kube-proxy or the equivalent field in the kube-proxy configuration file is set, <NodeIP> would be filtered node IP(s).

아래는 예시다:

``` yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: NodePort
  selector:
    app.kubernetes.io/name: MyApp
  ports:
      # By default and for convenience, the `targetPort` is set to the same value as the `port` field.
    - port: 80
      targetPort: 80
      # Optional field
      # By default and for convenience, the Kubernetes control plane will allocate a port from a range (default: 30000-32767)
      nodePort: 30007
```

### Type LoadBalancer
외부 load balancer를 제공하는 cloud provider에 대해 .spec.type을 LoadBalancer로 설정하면 svc에 대한 load balancer가 프로비저닝 된다. 실제 load balancer는 비동기적으로 생성된다. 그리고 해당 load balancer의 정보는 svc의 .status.loadBalancer 필드에 설정된다. 아래는 예시다:

``` yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app.kubernetes.io/name: MyApp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
  clusterIP: 10.0.171.239
  type: LoadBalancer
status:
  loadBalancer:
    ingress:
    - ip: 192.0.2.127
```

외부 load balancer에서 수신된 트래픽은 po로 라우팅된다. cloud provider가 로드 밸런싱을 결정한다.

일부 cloud provider는 .spec.loadBalancerIP 필드를 명시하는 것을 허용한다. 이 경우, 해당 ip를 사용하는 load balancer가 생성된다. loadBalancerIP가 명시되지 않으면 임시 ip가 할당된다. 해당 기능을 지원하지 않는 cloud provider에 대해 loadBalancerIP를 설정하더라도 해당 필드는 무시된다.

기본적으로 LoadBalancer 타입의 k8s svc는 kube-controller-manager 또는 cloud-controller-manager(in-tree controller라고도 함)의 CloudProvider 구성 요소에 내장된 k8s controller에 의해 조정(reconile)된다.

LoadBalancer Controller(LBC)가 LoadBalancer 타입의 k8s svc를 조정하도록 하기 위해 in-tree controller -> LBC로 조정을 오프로드해야한다.

해당 내용으 [관편 페이지](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.5/guide/service/nlb/#configuration)를 참고한다.

#### Load balancers with mixed protocol types
기본적으로 여러 포트가 정의된 경우 모든 포트는 동일한 프로토콜이어야 하며 해당 프로토콜은 cloud provider가 지원해야 한다.

MixedProtocolLBService feature gate(kube-apiserver v1.24에서 기본 활성화)는 여러 포트가 다른 프로토콜을 사용하는 것을 지원한다.

**Note**: cloud provider가 여러 프로토콜 사용을 지원하지 않는 경우 단일 프로토콜만 사용해야 한다.

#### Disabling load balancer NodePort allocation
.spec.allocateLoadBalancerNodePorts을 false로 설정해 no의 포트 할당을 비활성화 할 수 있다. no의 포트를 사용하는 대신 트래픽을 po로 직접 라우팅하도록 구현된 load balancer에서만 사용해야 한다. 기본적으로 .spec.allocateLoadBalancerNodePorts는 true이기 때문에 no의 포트를 할당한다. no의 포트가 할당된 svc에서 spec.allocateLoadBalancerNodePorts를 false로 변경하면 해당 no 포트의 할당이 자동으로 해제되지 않는다. no 포트의 할당을 해제하기 위해 svc의 .spec.ports[*].nodePorts 필드를 명시적으로 제거해야 한다.

#### Specifying class of load balancer implementation
cloud provider의 기본 load balancer가 아닌 다른 유형을 사용하길 원할 경우 .spec.loadBalancerClass 필드를 사용한다. 클러스터가 --cloud-provider flag를 사용해 cloud provider를 사용하도록 설정된 경우 .spec.loadBalancerClass의 기본값은 nil이며 cloud provider의 기본 로드 밸런서 구현을 사용한다. 

.spec.loadBalancerClass가 지정된 경우 지정된 클래스와 일치하는 로드 밸런서 구현이 해당 svc를 처리해야 한다. 기본 로드 밸런서 구현의 경우 해당 필드가 설정된 svc는 처리하지 않고 무시한다. spec.loadBalancerClass는 LoadBalancer 타입의 svc만 설정할 수 있는 필드다. 해당 필드는 변경이 불가하다. .spec.loadBalancerClass 값은 .metadata.label 스타일 지시자로 "internal-vjl" 또는 "example.com/internal-vip "과 같은 접두사가 있어야 한다. 접두사가 없는 것을 사용자를 위해 예약된 값이다.

#### Internal load balancer
동일한 vip 네트워크 환경에서 svc로부터의 트래픽을 라우팅해야 할 경우도 있다.

split-horizon DNS 환경에서 svc에 대해 외부, 내부 트래픽이 모두 라우팅될 수 있도록 해야 한다.

내부 load balancer를 설정하기 위해 cloud provider에 따른 annotation을 추가해야 한다. 아래는 aws 환경이다:

``` yaml
[...]
metadata:
    name: my-service
    annotations:
        service.beta.kubernetes.io/aws-load-balancer-internal: "true"
[...]
```

#### TLS support on AWS
AWS에서 실행되는 클러스터에서 부분적인 TLS/SSL 지원을 위해 다음과 같은 세 가지 annotation을 추가할 수 있다:

``` yaml
metadata:
  name: my-service
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-ssl-cert: arn:aws:acm:us-east-1:123456789012:certificate/12345678-1234-1234-1234-123456789012
```

1. 인증서의 ARN을 명시한다. 이는 third party 발급 되어 IAM에 업로드 된 인증서 또는 AWS Certificate Manager를 통해 발급된 인증서일 수 있다.

``` yaml
metadata:
  name: my-service
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-backend-protocol: (https|http|ssl|tcp)
```

2. po의 프로토콜을 지정한다. HTTPS, SSL의 경우 ELB는 인증서를 사용해 암호화된 연결을 통해 po가 자체 인증할 것으로 예상한다.


    HTTP, HTTPS는 7 계층 프록시를 선택한다: 요청을 전달할 때 ELB는 사용자와의 연결을 종료하고 http header를 분석하며 X-Forwarded-For 헤더에 클라이언트 IP를 주입한다(po는 ELB의 ip 주소를 보게된다)해 요청을 포워딩한다.
    
    TCP, SSL은 4 계층 프록시를 선택한다. ELB는 헤더를 수정하지 않고 트래픽을 전달한다.

일부 포트는 암호화, 일부 포트는 암호화하지 않고 사용할 경우 아래 annotation을 사용할 수 있다:

``` yaml
metadata:
  name: my-service
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-backend-protocol: http
    service.beta.kubernetes.io/aws-load-balancer-ssl-ports: "443,8443"
```

위 예시에서 svc는 80, 443, 8443 port를 사용하며 8443, 443 포트에 대해서 SSL 인증서를 사용, 80 포트에 대해서는 HTTP를 사용한다.

3. policy를 지정한다.

``` yaml
metadata:
  name: my-service
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-ssl-negotiation-policy: "ELBSecurityPolicy-TLS-1-2-2017-01"
```

k8s v1.9 이상부터 svc의 HTTP, SSL listerner를 위한 [predefined AWS SSL policies](https://docs.aws.amazon.com/elasticloadbalancing/latest/classic/elb-security-policy-table.html)를 사용할 수 있다. 사용 가능한 정책은 aws cli 명령어로 조회 가능하다:

``` bash
aws elb describe-load-balancer-policies --query 'PolicyDescriptions[].PolicyName'
```

#### PROXY protocol support on AWS

#### Connection Draining on AWS

#### Other ELB annotations

#### Network Load Balancer support on AWS
AWS의 NLB를 사용하기 위해 service.beta.kubernetes.io/aws-load-balancer-type annotation을 nlb 값을 사용한다.

``` yaml
metadata:
  name: my-service
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
```

**Note**: NLB는 특정 인스턴스 클래스에 대해 동작한다. 관련해서는 [AWS documentation](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/target-group-register-targets.html#register-deregister-targets)을 참고한다.

Classic Elastic Load Balancer와 다르게 NLB는 클라이언트 ip를 no에 전달한다. svc의 .spec.externalTrafficPolicy이 Cluster로 설정되면, 클라이언트 ip는 po에 전달되지 않는다.

.spec.externalTrafficPolicy을 Local로 설정하면 클라이언트 ip는 po에 전달된다. 하지만 균일하지 않은 트래픽 분산을 야기할 수도 있다. Nodes without any Pods for a particular LoadBalancer Service will fail the NLB Target Group's health check on the auto-assigned .spec.healthCheckNodePort and not receive any traffic.

이를 해결하기 위해 ds를 사용하거나 pod anti-affinity를 사용해 동일 no에 po를 배치시키지 않을 수 있다.

internal load balancer annotation을 NLB svc에 사용할 수 있다.

클라이언트의 트래픽이 NLB 뒤 po에 전달되기 위해 no의 security group은 아래 ip 규칙을 따라 수정되어야 한다.

| Rule           | Protocol | Port(s)                                                                         | IpRange(s)                                             | IpRange Description                              |
|----------------|----------|---------------------------------------------------------------------------------|--------------------------------------------------------|--------------------------------------------------|
| Health Check   | TCP      | NodePort(s) (.spec.healthCheckNodePort for .spec.externalTrafficPolicy = Local) | Subnet CIDR                                            | kubernetes.io/rule/nlb/health=\<loadBalancerName> |
| Client Traffic | TCP      | NodePort(s)                                                                     | .spec.loadBalancerSourceRanges (defaults to 0.0.0.0/0) | kubernetes.io/rule/nlb/client=\<loadBalancerName> |
| MTU Discovery  | ICMP     | 3,4                                                                             | .spec.loadBalancerSourceRanges (defaults to 0.0.0.0/0) | kubernetes.io/rule/nlb/mtu=\<loadBalancerName>    |

nlb에 접근하는 클라이언트 ip를 제한하기 위해 .spec.loadBalancerSourceRanges 필드를 사용할 수 있다.

``` yaml
spec:
  loadBalancerSourceRanges:
    - "143.231.0.0/16"
```

**Note**: .spec.loadBalancerSourceRanges이 설정되지 않으면 k8s는 0.0.0.0/0에서 전달되는 트래픽을 no의 security group으로 전달한다. 만약 no가 공용 ip를 갖는 경우, nlb가 아닌 트래픽도 security group의 모든 인스턴스에 접근할 수 있다.

Further documentation on annotations for Elastic IPs and other common use-cases may be found in the AWS Load Balancer Controller documentation.

#### Other CLB annotations on Tencent Kubernetes Engine (TKE)


### Type ExternalName
ExternalName 타입의 svc는 svc를 도메인에 매핑한다. 해당 타입의 svc는 .spec.externalNam를 명시해야 한다. 예를 들어 아래 my-service svc는 my.database.example.com에 매핑된다:

``` yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
  namespace: prod
spec:
  type: ExternalName
  externalName: my.database.example.com
```

**Note**: ExternalName accepts an IPv4 address string, but as a DNS name comprised of digits, not as an IP address. ExternalNames that resemble IPv4 addresses are not resolved by CoreDNS or ingress-nginx because ExternalName is intended to specify a canonical DNS name. ip 주소를 지정하고자 할 경우에는 headless svc 사용을 고려해야 한다.

**Warning**: HTTP, HTTPS를 포함한 일부 일반 프로토콜에 대해 externalName을 사용하는 데 문제가 있을 수 있다. externalName을 사용하는 경우 클러스터 내의 클라이언트에서 사용하는 호스트네임이 externalName이 참조하는 이름과 다르다.

호스트네임을 사용하는 프로토콜의 경우 이 차이로 인해 오류가 발생하거나 예기치 않은 응답이 발생할 수 있다. HTTP 요청에는 origin server 인식하지 못하는 Host: 헤더가 있다. TLS 서버는 클라이언트가 연결된 호스트 이름과 일치하는 인증서를 제공할 수 없다.

**Note**: This section is indebted to the Kubernetes Tips - Part 1 blog post from Alen Komljen.

### External IPs
클러스터의 no로 라우팅이 필요한 외부 ip가 존재하는 경우 k8s svc는 .spec.externalIPs 필드를 사용해 노출될 수 있다. external ip, svc port로 인입되는 트래픽은 svc의 엔드포인트로 라우팅된다. 사용자는 해당 ip를 갖는 트래픽이 no에 잘 도착할 수 있도록 책임져야한다. 일반적인 사용 예시는 k8s 시스템의 일부가 아닌 외부 로드 밸런서를 사용하는 경우다.

.spec.externalIPs 필드는 어떤 svc 타입과도 사용될 수 있다. 아래 예시에서 my-service svc에 대해 클라이언트는 80.11.12.10:80(externalIP:port)를 통해 접근할 수 있다.

``` yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app.kubernetes.io/name: MyApp
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 9376
  externalIPs:
    - 80.11.12.10
```

## Shortcomings
Using the userspace proxy for VIPs works at small to medium scale, but will not scale to very large clusters with thousands of Services. The original design proposal for portals has more details on this.

Using the userspace proxy obscures the source IP address of a packet accessing a Service. This makes some kinds of network filtering (firewalling) impossible. The iptables proxy mode does not obscure in-cluster source IPs, but it does still impact clients coming through a load balancer or node-port.

The Type field is designed as nested functionality - each level adds to the previous. This is not strictly required on all cloud providers (e.g. Google Compute Engine does not need to allocate a NodePort to make LoadBalancer work, but AWS does) but the current API requires it.

## Virtual IP implementation
이전 내용을 통해 대부분의 사람들이 svc를 이해하는 데 충분히 도움이된다. 하지만 관련된 상세 사항을 이해하면 더욱 도움이 될 수 있다.

### Avoiding collisions

### Service IP addresses

## API Object
Service is a top-level resource in the Kubernetes REST API. You can find more details about the Service API object.

## Supported protocols
### TCP
모든 svc 타입에 대해 TCP를 사용할 수 있으며 기본 값이다.

### UDP
대부분의 svc 타입에 UDP를 사용할 수 있다. type-LoadBalancer svc의 경우 사용 중인 cloud provier의 기능 지원 여부에 따라 다르다.

### SCTP
SCTP 트래픽을 지원하는 네트워크 플러그인을 사용할 때, 대부분의 svc에 대해 SCTP를 사용할 수 있다. type=LoadBalancer svc에 대해 SCTP 지원은 cloud provier의 기능 지원 여부에 따라 다르다.

### HTTP
사용 중인 cloud provider가 지원하는 경우, svc를 LoadBalancer 모드로 사용해 외부 HTTP / HTTPS를 svc의 ep로 리버스 프록싱할 수 있다.

**Note**: svc 대신 HTTP / HTTPS svc를 노출하기 위해 ingress를 사용할 수도 있다.

### PROXY protocol
If your cloud provider supports it, you can use a Service in LoadBalancer mode to configure a load balancer outside of Kubernetes itself, that will forward connections prefixed with PROXY protocol.

The load balancer will send an initial series of octets describing the incoming connection, similar to this example

```
PROXY TCP4 192.0.2.202 10.0.42.7 12345 7\r\n
```

followed by the data from the client.

