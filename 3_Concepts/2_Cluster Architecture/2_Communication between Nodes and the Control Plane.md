## Node to Control Plane
k8s는 "hub-and-spoke" API 패턴을 사용한다. no와 po는 kube-apiserver의 API를 사용해 control plane과 통신한다. control plane의 다른 구성 요소는 API를 노출하지 않는다. kube-apiserver는 안전한 HTTPS 443 listen port를 사용하며 1가지 이상의 [authentication](https://kubernetes.io/docs/reference/access-authn-authz/authentication/)이 활성화 되어 있다. 그리고 [anonymous request](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#anonymous-requests), [service account token](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#service-account-tokens)이 허용된 경우 1가지 이상의 [authorization](https://kubernetes.io/docs/reference/access-authn-authz/authorization/)이 활성화 되어 있어야 한다.

no는 cluster의 public root certificate가 provision 되어야 한다. 이를 통해 no는 유효한 client credential을 사용해 안전하게 kube-apiserver에 연결할 수 있다. kubelet에 제공되는 client credentials은 client certificate 형식이 권장된다. kubelet client certificate의 자동 provision은 [kubelet TLS bootstraping](https://kubernetes.io/docs/reference/access-authn-authz/kubelet-tls-bootstrapping/)을 참고한다.

po는 sa를 사용해 kube-apiserver에 안전하게 연결할 수 있다. 이를 위해 k8s는 po가 생성될 때 public root certificate와 유효한 bearer token을 자동으로 주입한다. default ns의 kubernetes svc는 kube-apiserver의 HTTPS endpoint로 redirect(kube-proxy가 수행)되는 virtual ip로 구성되어 있다.

control plane 구성 요소 역시 kube-apiserver와 안전한 포트를 사용해 통신한다.

결과적으로 no, po와 control plane의 연결은 기본적으로 안전하기 때문에 공용 네트워크에서 실행할 수 있다.

## Control plane to node
control plane(kube-apiserver)은 no와 통신하기 위해 두 가지 방식을 사용한다. 첫 번째는 kube-apiserver가 각 no의 kubelet과 통신한다. 두 번째는 kube-apiserver의 proxy 기능을 통해 모든 no, po로 통신한다.

### API server to kubelet
kube-apiserver가 kubelet과 통신하는 이유는 다음과 같다.
- po의 로그를 가져온다.
- 실행 중인 po에 대한 attach(주로 kubectl 명령어를 사용)
- kubelet의 port-forwarding 기능 제공



### API server to nodes, pods, and services

### SSH tunnels

### Konnectivity service