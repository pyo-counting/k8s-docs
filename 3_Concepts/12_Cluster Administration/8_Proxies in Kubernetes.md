k8s에서 사용되는 proxy를 설명한다.

## Proxies
k8s를 사용하면서 마주칠 수 있는 몇 가지 다른 proxy가 있다.
1. [kubectl proxy](https://kubernetes.io/docs/tasks/access-application-cluster/access-cluster/#directly-accessing-the-rest-api)
    - 사용자 데스크탑 또는 po에서 실행한다.
    - localhost 주소를 kube-apiserver로 프록시한다.
    - clinet는 proxy에 HTTP를 사용한다.
    - proxy는 kube-apiserver에 HTTPS를 사용한다.
    - authentication header를 추가한다.
2. [apiserver proxy](https://kubernetes.io/docs/tasks/access-application-cluster/access-cluster-services/#discovering-builtin-services)
    - kube-apiserver에 내장된 bastion이다.
    - cluster 외부 사용자가 cluster 내부에 접근할 수 있도록 cluster IP에 연결한다.
    - kube-apiserver 프로세스 내에서 실행된다.
    - client는 proxy에 HTTPS(kube-apiserver가 HTTP를 사용하도록 설정해다면 HTTP)를 사용한다.
    - proxy to target may use HTTP or HTTPS as chosen by proxy using available information
    - no, po, svc에 접근하기 위해 사용된다.
    - svc에 접근할 때 load balancing를 수행한다.
3. [kube proxy](https://kubernetes.io/docs/concepts/services-networking/service/#ips-and-vips) 
    - 각 no에서 실행한다.
    - UDP, TCP, SCTP를 프록시한다.
    - HTTP를 이해하지 못한다.
    - load balancing를 지원한다.
    - svc에 접근하기 위한 용도로만 사용된다.
4. kube-apiserver 앞 단의 proxy/load balancer
    - 존재 여부, 구현은 cluster마다 다를 수 있다.
    - client와 kube-apiserver 사이에 존재한다.
    - kube-apiserver가 여럿일 경우 load balancing를 수행한다.
5. cloud load balancer
    - cloud provider가 제공한다.
    - svc가 `LoadBalancer` 타입일 경우 생성된다.
    - UDP, TCP만 지원한다.
    - SCTP는 cloud provier의 구현에 따라 다르다.
    - cloud provider에 따라 구현이 다르다.

## Requesting redirects
proxy는 redirection 기능을 대체했다. redirection은 deprecated다.