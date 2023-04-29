ingress는 클러스터 내 svc에대한 외부 접근(일반적으로 HTTP)을 관리하기 위한 k8s resource다.

ingress는 load balancing, SSL termintation, name-based virtual hosting 기능을 제공한다.

## Terminology
해당 문서에서는 아래와 같은 단어을 사용한다:
- Node: k8s 내에서 클러스터 일부로서 worker machine.
- Cluster: k8s에 의해 관리되는 컨테이너화된 애플리케이션을 실행하는 node의 집합. 대부분의 k8s 배포 환경에서 클러스터 내 no는 공용 인터넷의 일부가 아니다.
- Edge router: 클러스터에 방화벽 정책을 적용하는 라우터. 라우터는 cloud provider가 관리하는 게이트웨이 또는 실제 물리 장비일 수도 있다.
- Cluster network: k8s 네트워크 모델에 따라 클러스터 내에서 통신을 용이하게 하는 논리적 또는 물리적 link에 대한 집합이다.
- Service: k8s svc는 labele selctor를 사용해 po의 집합을 식별한다. svc는 클라우드 내 네트워크 환경에서만 라우팅이 가능한 가상 ip를 갖는다.

## What is Ingress?
ingress는 클러스터 내부 svc에 대한 클러스터 외부 HTTP, HTTPS route를 노출한다.

아래는 ingress가 1개의 svc로 트래픽을 전달하는 예시다:
![what is ingress?](https://d33wubrfki0l68.cloudfront.net/91ace4ec5dd0260386e71960638243cf902f8206/c3c52/docs/images/ingress.svg)

ingress를 통해 svc에 대한 외부 url 접근, 트래픽에 대한 load balacning, terminate SSL / TLS, name-based virtual hosting이 가능하다. ingress controller는 일반적으로 load balancer를 사용해 ingress를 역할을 구현하지만 edge router, 추가 frontend를 구성할 수 있다.

ingerss는 임의 프로토콜 / 포트를 노출하지 않는다. HTTP, HTTPS가 아닌 svc를 노출이 필요할 때는 일반적으로 .spec.type이 NodePort 또는 LoadBalancer인 svc를 사용한다.

## Prerequisites
ingress를 위한 ingress controller가 필요하다.

ingress-nginx와 같은 ingress controller를 배포해야 한다. 여러 ingress controller를 사용해도 된다.

이상적으로는 모든 ingress controller가 규격을 충족해야 하지만 실제로는 ingress controller마다 다르게 동작할 수도 있다.

**Note**: ingress controller의 documentation을 살펴보고 이러한 차이점과 주의 사항을 이해해야 한다.

## The Ingress resource
아래는 ingress 객체의 간단한 예시다:

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

### Ingress rules

### DefaultBackend

### Resource backends

### Path types

### Examples

## Hostname wildcards