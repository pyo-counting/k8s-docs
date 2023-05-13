ing는 클러스터 내 svc에대한 외부 접근(일반적으로 HTTP)을 관리하기 위한 k8s resource다.

ing는 load balancing, SSL termintation, name-based virtual hosting 기능을 제공한다. ingres는 요청 HTTP host 헤더, path에 따라 트래픽을 제어한다.

## Terminology
해당 문서에서는 아래와 같은 단어을 사용한다:
- Node: k8s 내에서 클러스터 일부로서 worker machine.
- Cluster: k8s에 의해 관리되는 컨테이너화된 애플리케이션을 실행하는 node의 집합. 대부분의 k8s 배포 환경에서 클러스터 내 no는 공용 인터넷의 일부가 아니다.
- Edge router: 클러스터에 방화벽 정책을 적용하는 라우터. 라우터는 cloud provider가 관리하는 게이트웨이 또는 실제 물리 장비일 수도 있다.
- Cluster network: k8s 네트워크 모델에 따라 클러스터 내에서 통신을 용이하게 하는 논리적 또는 물리적 link에 대한 집합이다.
- Service: k8s svc는 label selctor를 사용해 po의 집합을 식별한다. svc는 클라우드 내 네트워크 환경에서만 라우팅이 가능한 가상 ip를 갖는다.

## What is Ingress?
ing는 클러스터 내부 svc에 대한 클러스터 외부 HTTP, HTTPS route를 노출한다.

아래는 ingress가 1개의 svc로 트래픽을 전달하는 예시다:
![what is ingress?](https://d33wubrfki0l68.cloudfront.net/91ace4ec5dd0260386e71960638243cf902f8206/c3c52/docs/images/ingress.svg)

ing를 통해 svc에 대한 외부 url 접근, 트래픽에 대한 load balacning, terminate SSL / TLS, name-based virtual hosting이 가능하다. ingress controller는 일반적으로 load balancer를 사용해 ing를 역할을 구현하지만 edge router, 추가 frontend를 구성할 수 있다.

ingerss는 임의 프로토콜 / 포트를 노출하지 않는다. HTTP, HTTPS가 아닌 svc를 노출이 필요할 때는 일반적으로 .spec.type이 NodePort 또는 LoadBalancer인 svc를 사용한다.

## Prerequisites
ing를 위한 ingress controller가 필요하다.

ingress-nginx와 같은 ingress controller를 배포해야 한다. 여러 ingress controller를 사용해도 된다.

이상적으로는 모든 ingress controller가 규격을 충족해야 하지만 실제로는 ingress controller마다 다르게 동작할 수도 있다.

**Note**: ingress controller의 documentation을 살펴보고 이러한 차이점과 주의 사항을 이해해야 한다.

## The Ingress resource
아래는 ing 객체의 간단한 예시다:

``` yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: minimal-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx-example
  rules:
  - http:
      paths:
      - path: /testpath
        pathType: Prefix
        backend:
          service:
            name: test
            port:
              number: 80
```

ing는 이름은 유효한 DNS subdomain name 규칙을 따라야 한다. ing는 ingress controller에 따른 설정을 위해 annotation을 사용한다. 각 ingress controller마다 다른 annotation을 사용할 수 있다.

ing의 .spec에는 load balancer 또는 porxy server 설정을 위한 정보가 있다. 인입 요청에 대한 라우팅을 위한 규칙에 대한 목록도 포함한다. ing는 HTTP, HTTPS에 대한 트래픽에 대한 규칙만 지원한다.

.spec.ingressClassName이 생략된다면 기본 설정 ingress class가 정의되어 있어야한다.

기본 IngressClass resource 없이 동작하는 ingress controller도 존재한다. 예를 들어 Ingress-NGINX controller는 --watch-ingress-without-class flag를 통해 설정될 수 있다. 물론 [아래](https://kubernetes.io/docs/concepts/services-networking/ingress/#default-ingress-class)와 같이 기본 IngressClass를 지정하는 것

### Ingress rules
각 HTTP 규칙(.spec.rules)은 아래 정보를 포함한다:

- .spec.rules[*].host: 옵션 값. 설정하지 않으면 규칙이 지정된 IP 주소를 통과하는 모든 인바운드 HTTP 트래픽에 적용된다. 설정하면 해당 host에 대해서만 규칙이 적용된다.
- .spec.rules[\*].http.paths[\*]: 각 path는 .spec.rules[\*].http.paths[*].service를 통해 정의된 backend와 관련이 있다. host, path가 요청과 일치하는지 여부를 확인하고 load balancer가 svc로 트래픽을 보낸다.
- backend는 svc 또는 CRD를 통해 설정할 수 있다. ingress에 대한 HTTP 요청에 대해 host, path가 일치하는 backend로 요청을 보낸다.

.spec.defaultBackend을 설정해 path와 일치하지 않는 요청을 다루는데 사용한다.

### DefaultBackend
규칙이 없는 ing는 모든 트래픽을 .spec.defaultBackend로 전송한다. defaultBackend는 일반적으로 ingress controller의 설정 옵션이고 and is not specified in your Ingress resources. .spec.rules이 없으면 .spec.defaultBackend은 반드시 정의되어야 한다. defaultBackend가 설정되지 않은 경우 규칙과 일치하지 않는 요청에 대한 처리는 ingress controller의 몫이다(관련해 사용하는 ingress controller의 공식 문서를 참고해야 한다).

ing 리소스에 대해 HTTP 요청이 어떠한 룰에도 매칭되지 않는 경우 기본 backend로 라우팅된다.

### Resource backends
svc가 아닌 resource 타입의 backend는 해당 ing와 동일한 ns에 존재하는 k8s 리소스를 가리키기 위한 [ObjectRef](https://kubernetes.io/docs/reference/kubernetes-api/common-definitions/typed-local-object-reference/#TypedLocalObjectReference)다. backend에 대해 resource 타입과 svc에 대한 정의는 상호 배타적이기 때문에 둘 다 모두 정의된 경우 유효성 검사에 대해 실패한다. resource backend에 대한 일반적인 용도는 static asset을 위한 object storage를 가리키는 것이다.

``` yaml
kind: Ingress
metadata:
  name: ingress-resource-backend
spec:
  defaultBackend:
    resource:
      apiGroup: k8s.example.com
      kind: StorageBucket
      name: static-assets
  rules:
    - http:
        paths:
          - path: /icons
            pathType: ImplementationSpecific
            backend:
              resource:
                apiGroup: k8s.example.com
                kind: StorageBucket
                name: icon-assets
```

위 ing에 대해 아래 명령어를 통해 조회 가능하다.

``` bash
$ kubectl describe ingress ingress-resource-backend

Name:             ingress-resource-backend
Namespace:        default
Address:
Default backend:  APIGroup: k8s.example.com, Kind: StorageBucket, Name: static-assets
Rules:
  Host        Path  Backends
  ----        ----  --------
  *
              /icons   APIGroup: k8s.example.com, Kind: StorageBucket, Name: icon-assets
Annotations:  <none>
Events:       <none>
```

### Path types
.spec.rules[*].http.paths[*].pathType는 필수 설정이다. 세 가지 설정 값을 사용할 수 있다.

- ImplementationSpecific: IngressClass resource에 따라 다르다. 별도의 pathType으로 처리허거나 Prefix, Exact와 동일하게 처리될 수도 있다.
- Exact: URL path가 정확하게 매치(대소문자 구별)
- Prefix: path 접두사인 /로 구분되는 구성요소에 대해 매치. Matching is case sensitive and done on a path element by element basis. A path element refers to the list of labels in the path split by the / separator. A request is a match for path p if every p is an element-wise prefix of p of the request path.

**Note**: path의 마지막 구성요소가 요청 path의 마지막 구성요소의 일부 문자열과 일치할 경우 이는 매칭되지 않는 것이다(예를 들어 /foo/bar는 /foo/bar/baz에 매칭되지만 /foo/barbaz에는 매칭되지 않는다).

### Examples
| Kind   | Path(s)                     | Request path(s) | Matches?                         |
|--------|-----------------------------|-----------------|----------------------------------|
| Prefix | /                           | (all paths)     | Yes                              |
| Exact  | /foo                        | /foo            | Yes                              |
| Exact  | /foo                        | /bar            | No                               |
| Exact  | /foo                        | /foo/           | No                               |
| Exact  | /foo/                       | /foo            | No                               |
| Prefix | /foo                        | /foo, /foo/     | Yes                              |
| Prefix | /foo/                       | /foo, /foo/     | Yes                              |
| Prefix | /aaa/bb                     | /aaa/bbb        | No                               |
| Prefix | /aaa/bbb                    | /aaa/bbb        | Yes                              |
| Prefix | /aaa/bbb/                   | /aaa/bbb        | Yes, ignores trailing slash      |
| Prefix | /aaa/bbb                    | /aaa/bbb/       | Yes, matches trailing slash      |
| Prefix | /aaa/bbb                    | /aaa/bbb/ccc    | Yes, matches subpath             |
| Prefix | /aaa/bbb                    | /aaa/bbbxyz     | No, does not match string prefix |
| Prefix | /, /aaa                     | /aaa/ccc        | Yes, matches /aaa prefix         |
| Prefix | /, /aaa, /aaa/bbb           | /aaa/bbb        | Yes, matches /aaa/bbb prefix     |
| Prefix | /, /aaa, /aaa/bbb           | /ccc            | Yes, matches / prefix            |
| Prefix | /aaa                        | /ccc            | No, uses default backend         |
| Mixed  | /foo (Prefix), /foo (Exact) | /foo            | Yes, prefers Exact               |


#### Multiple matches
요청이 ing의 여러 rule에 매칭될 수도 있다. 이 경우 매칭되는 path가 가장 긴 rule에 대해 우선 순위가 높다. 만약 이에 대해서도 정확하게 일치한다면 exact pathType이 prefix pathType보다 우선 순위가 높다.

## Hostname wildcards
host는 완전(예를 들어, foo.bar.com) 매칭 또는 wildcard(예를 들어, \*.bar.com)를 사용할 수도 있다. 완전 매칭의 경우 HTTP Host header가 .spec.rules[*].host와 완전히 일치해야 한다. wildcard 매치의 경우 HTTP Host 헤더가 wildcard 규칙의 접미사와 일치해야 한다.

| Host      | Host header     | Match?                                            |
|-----------|-----------------|---------------------------------------------------|
| *.foo.com | bar.foo.com     | Matches based on shared suffix                    |
| *.foo.com | baz.bar.foo.com | No match, wildcard only covers a single DNS label |
| *.foo.com | foo.com         | No match, wildcard only covers a single DNS label |

``` yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-wildcard-host
spec:
  rules:
  - host: "foo.bar.com"
    http:
      paths:
      - pathType: Prefix
        path: "/bar"
        backend:
          service:
            name: service1
            port:
              number: 80
  - host: "*.foo.com"
    http:
      paths:
      - pathType: Prefix
        path: "/foo"
        backend:
          service:
            name: service2
            port:
              number: 80
```

## Ingress class
ing는 여러 여러 다른 controller에 의해 구현된다. 각 ingress는 class를 명시해야 한다, 명시된 ingress class 리소스는 해당 ing에 대한 추가 설정을 포함한다.

``` yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: external-lb
spec:
  controller: example.com/ingress-controller
  parameters:
    apiGroup: k8s.example.com
    kind: IngressParameters
    name: external-lb
```

IngressClass의 .spec.parameters는 추가 설정을 포함하는 다른 resource를 가리킨다.

파라미터는 .spec.controller를 통해 지정되는 controller에 따라 달라질 수 있다.

### IngressClass scope
ingress controller에 따라 cluster-wide 또는 ns 범위의 파라미터를 사용할 수 있다.

### Cluster
IngressClass 파라미터의 기본 범위는 cluster-wide이다.

.spec.parameters.scope를 지정하지 않거나 Cluster로 값을 지정하면 IngressClass는 cluster-scoped 리소스를 참조한다. The kind (in combination the apiGroup) of the parameters refers to a cluster-scoped API (possibly a custom resource), and the name of the parameters identifies a specific cluster scoped resource for that API.

예를 들어:

``` yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: external-lb-1
spec:
  controller: example.com/ingress-controller
  parameters:
    # The parameters for this IngressClass are specified in a
    # ClusterIngressParameter (API group k8s.example.net) named
    # "external-config-1". This definition tells Kubernetes to
    # look for a cluster-scoped parameter resource.
    scope: Cluster
    apiGroup: k8s.example.net
    kind: ClusterIngressParameter
    name: external-config-1
```

### Namespaced
.spec.parameters.scope를 Namespace 값을 지정하면 IngressClass는 namespaced-scopre 리소스를 참조한다. .spec.parameters.namespace 필드를 사용해 파라미터가 속한 ns도 지정해야 한다.

The kind (in combination the apiGroup) of the parameters refers to a namespaced API (for example: ConfigMap), and the name of the parameters identifies a specific resource in the namespace you specified in namespace.

Namespace-scoped parameters help the cluster operator delegate control over the configuration (for example: load balancer settings, API gateway definition) that is used for a workload. If you used a cluster-scoped parameter then either:

the cluster operator team needs to approve a different team's changes every time there's a new configuration change being applied.
the cluster operator must define specific access controls, such as RBAC roles and bindings, that let the application team make changes to the cluster-scoped parameters resource.
The IngressClass API itself is always cluster-scoped.

Here is an example of an IngressClass that refers to parameters that are namespaced:

``` yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: external-lb-2
spec:
  controller: example.com/ingress-controller
  parameters:
    # The parameters for this IngressClass are specified in an
    # IngressParameter (API group k8s.example.com) named "external-config",
    # that's in the "external-configuration" namespace.
    scope: Namespace
    apiGroup: k8s.example.com
    kind: IngressParameter
    namespace: external-configuration
    name: external-config
```

### Deprecated annotation
IngressClass resource와 ing resource의 .spec.ingressClassName 필드가 추가되기 전(k8s 1.18)에는 kubernetes.io/ingress.class annotation을 사용했다. 이 annotation은 공식적으로 정의된 것은 아니였지만 여러 ingress controller에서 사용했다.

.spec.ingresClassName은 해당 annotation에 대한 대체 필드지만 완전하게 동일하지는 않다. annotation은 일반적으로 ingress를 구현하는 ingress controller의 이름을 참조하는 데 사용됐지만, 필드는 ingress controller의 이름, 추가 ingress에 대한 설정을 포함하는 IngressClass resource를 참조하는 데 사용된다.

### Default IngressClass
cluster에 대한 기본 ingress controller를 설정할 수 있다. ingressclass.kubernetes.io/is-default-class annotation에 대해 true로 설정하면 된다. .spec.ingressClassName 필드가 없는 ing에 대해 기본 ingress controller가 사용된다.

**Caution**: cluster 내에 여러 기본 ingress controller가 있을 경우 ingressClassName가 없는 ingress에 대해 admission controller가 새로운 ing object 생성을 막는다. 이를 막기 위해 1개의 기본 ingress controller만 지정해야 한다.

기본 IngressClass 정의 없이 동작하는 ingress controller도 있다. 예를 들어 Ingress-NGINX controller의 경우 --watch-ingress-without-class flag를 통해 지정이 가능하다. 하지만 기본 IngressClass를 명시하는 것을 권장한다:

``` yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  labels:
    app.kubernetes.io/component: controller
  name: nginx-example
  annotations:
    ingressclass.kubernetes.io/is-default-class: "true"
spec:
  controller: k8s.io/ingress-nginx
```

## Types of Ingress
### Ingress backed by a single Service
단일 svc를 노출할 수 있는 기존 k8s이 있다. rule이 없는 default backend를 지정해 이 작업을 수행할 수도 있다.

``` yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: test-ingress
spec:
  defaultBackend:
    service:
      name: test
      port:
        number: 80
```

아래 명령어를 사용해 위 yaml을 통해 생성된 ing를 조회할 수 있다.

``` bash
kubectl get ingress test-ingress

NAME           CLASS         HOSTS   ADDRESS         PORTS   AGE
test-ingress   external-lb   *       203.0.113.123   80      59s
```

203.0.113.123는 ingress controller가 할당한 IP다.

**Note**: ingress controller, load balacner IP 주솔르 할당하는 데 조금 시간이 걸릴 수 있다. 이 때까지 ADDRESS 필드에 대해 \<pending> 값이 보인다.

### Simple fanout
일반적인 구성은 요청 HTTP URI에 따라 둘 이상의 svc로 라우팅하는 경우다. ingress를 이용하면 load balancer의 수를 최소한으로 관리할 수 있다. 아래는 예시다:

![](https://d33wubrfki0l68.cloudfront.net/36c8934ba20b97859854610063337d2072ea291a/28e8b/docs/images/ingressfanout.svg)

``` yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: simple-fanout-example
spec:
  rules:
  - host: foo.bar.com
    http:
      paths:
      - path: /foo
        pathType: Prefix
        backend:
          service:
            name: service1
            port:
              number: 4200
      - path: /bar
        pathType: Prefix
        backend:
          service:
            name: service2
            port:
              number: 8080
```

아래 명령어를 사용해 위 yaml을 통해 생성된 ing를 조회할 수 있다.

``` bash
kubectl describe ingress simple-fanout-example

Name:             simple-fanout-example
Namespace:        default
Address:          178.91.123.132
Default backend:  default-http-backend:80 (10.8.2.3:8080)
Rules:
  Host         Path  Backends
  ----         ----  --------
  foo.bar.com
               /foo   service1:4200 (10.8.0.90:4200)
               /bar   service2:8080 (10.8.0.91:8080)
Events:
  Type     Reason  Age                From                     Message
  ----     ------  ----               ----                     -------
  Normal   ADD     22s                loadbalancer-controller  default/test
```

**Note**: 사용하는 ingress controller에 따라 default-http-backend svc를 생성해야 될 수도 있다.

### Name based virtual hosting
Name-based virtual hosts는 동일 IP에 대해 여러 host에 대한 HTTP 트래픽을 제어할 수 있다.

![](https://d33wubrfki0l68.cloudfront.net/638e6be4c880b8497c7abd899c2bb0f972389c04/55a48/docs/images/ingressnamebased.svg)

``` yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: name-virtual-host-ingress
spec:
  rules:
  - host: foo.bar.com
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: service1
            port:
              number: 80
  - host: bar.foo.com
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: service2
            port:
              number: 80
```

rule에 host가 없는 ing resource를 생성할 경우 name based virtual host를 필요로 하지 않고 ingress ip에 대한 모든 웹 트래픽을 매칭시킬 수 있다.

예를 들어 아래 ing는 first.bar.com에 대한 트래픽은 service1, second.bar.com에 대한 트래픽은 service2, 앞의 두 host에 매칭되지 않는 모든 트래픽은 service3로 보낸다.

``` yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: name-virtual-host-ingress-no-third-host
spec:
  rules:
  - host: first.bar.com
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: service1
            port:
              number: 80
  - host: second.bar.com
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: service2
            port:
              number: 80
  - http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: service3
            port:
              number: 80
```

### TLS
TLS private key, certiface를 초함하는 secret resource를 ing에 명시할 수 있다. ing resource는 단일 TLS port 443만 지원한다(svc, po에 대한 트래픽은 HTTP). ing의 TLS가 서로 다른 호스트를 지정할 때 ingress controller가 SNI를 지원하는 경우 SNI TLS extension을 통해 지정된 호스트 이름에 다라 동일한 포트에서 다중화한다. TLS 암호화에는 tls.crt, tls.key 이름의 key가 포함된 secert이 참조되어야 한다. 

``` yaml
apiVersion: v1
kind: Secret
metadata:
  name: testsecret-tls
  namespace: default
data:
  tls.crt: base64 encoded cert
  tls.key: base64 encoded key
type: kubernetes.io/tls
```

ing resource는 클라이언트 <-> load balancer 구간을 TLS를 사용해 암호화한다. certificate는 CN(Common Name), FQDN(Fully Qualified Domain Name)을 포함해야 한다.

**Note**: certificate는 모든 sub domain을 위해 발행되기 때문에 기본 rule에 대해서는 TLS가 동작하지 않음을 명시해야 한다. 그러므로 .spec.tls[*].hosts[*]은 .spec.rules[*].host와 일치해야 한다.

``` yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-example-ingress
spec:
  tls:
  - hosts:
      - https-example.foo.com
    secretName: testsecret-tls
  rules:
  - host: https-example.foo.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: service1
            port:
              number: 80
```

**Note**: ingree controller에 따라 TLS 기능에 대한 차이가 있을 수 있다. 각 ingress controller를 살펴보고 TLS에 대한 실제 동작 방식을 이해해야 한다.

### Load balancing
ingress controller는 load balacning 알고리즘, backend weight scheme 등 모든 ingress에 적용되는 load balacning 정책을 갖고 부트스트랩된다. 보다 전문적인 load balancing 개념(예를 들어 persistent session, dynamic weight)은 아직 ingress에 노출되지 않는다. 대신 svc에 대한 load balancer를 통해 이러한  기능을 얻을 수 있다.

health check가 ing resource를 통해 직접 노출되진 않지만 동일한 기능을 위한 readiness probe도 있다. health check에 대한 구현은 각 ingress controller를 참조하면 된다.

## Updating an Ingress
새로운 host를 추가하기 위해 ing reousrce를 업데이트할 수 있다.

``` bash
kubectl describe ingress test

Name:             test
Namespace:        default
Address:          178.91.123.132
Default backend:  default-http-backend:80 (10.8.2.3:8080)
Rules:
  Host         Path  Backends
  ----         ----  --------
  foo.bar.com
               /foo   service1:80 (10.8.0.90:80)
Annotations:
  nginx.ingress.kubernetes.io/rewrite-target:  /
Events:
  Type     Reason  Age                From                     Message
  ----     ------  ----               ----                     -------
  Normal   ADD     35s                loadbalancer-controller  default/test

kubectl edit ingress test

spec:
  rules:
  - host: foo.bar.com
    http:
      paths:
      - backend:
          service:
            name: service1
            port:
              number: 80
        path: /foo
        pathType: Prefix
  - host: bar.baz.com
    http:
      paths:
      - backend:
          service:
            name: service2
            port:
              number: 80
        path: /foo
        pathType: Prefix
```

위와 같이 kubectl edit 명령어를 사용해 test 이름을 갖는 ing resource를 업데이트 및 저장함으로써 ingress controller가 load balancer에 대한 변경 사항을 업데이트 한다.

``` bash
kubectl describe ingress test

Name:             test
Namespace:        default
Address:          178.91.123.132
Default backend:  default-http-backend:80 (10.8.2.3:8080)
Rules:
  Host         Path  Backends
  ----         ----  --------
  foo.bar.com
               /foo   service1:80 (10.8.0.90:80)
  bar.baz.com
               /foo   service2:80 (10.8.0.91:80)
Annotations:
  nginx.ingress.kubernetes.io/rewrite-target:  /
Events:
  Type     Reason  Age                From                     Message
  ----     ------  ----               ----                     -------
  Normal   ADD     45s                loadbalancer-controller  default/test
```

kubectl replace -f 명령어를 사용해 동일한 결괄르 얻을 수 있다.

## Failing across availability zones
failure domain 간 트래픽을 분산시키는 기술은 cloud provider마다 다를 수 있다. 자세한 내용은 ingress controller의 문서를 참고한다.