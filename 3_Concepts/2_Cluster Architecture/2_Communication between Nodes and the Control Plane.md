## Node to Control Plane
k8s는 "hub-and-spoke" API 패턴을 사용한다. no와 po는 kube-apiserver의 API를 사용해 control plane과 통신한다. control plane의 다른 구성 요소는 API를 노출하지 않는다. kube-apiserver는 안전한 HTTPS 443 listen port를 사용하며 1가지 이상의 [authentication](https://kubernetes.io/docs/reference/access-authn-authz/authentication/)이 활성화 되어 있다. 그리고 [authorization](https://kubernetes.io/docs/reference/access-authn-authz/authorization/)이 활성화 되어있어야 한다. 특히 [anonymous request](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#anonymous-requests), [service account token](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#service-account-tokens)을 사용하는 경우

no는 cluster의 public root certificate가 provision 되어야 한다. 이를 통해 no는 유효한 client credential을 사용해 안전하게 kube-apiserver에 연결할 수 있다. kubelet에 제공되는 client credentials은 client certificate 형식이 권장된다. kubelet client certificate의 자동 provision은 [kubelet TLS bootstraping](https://kubernetes.io/docs/reference/access-authn-authz/kubelet-tls-bootstrapping/)을 참고한다.

po는 sa를 사용해 kube-apiserver에 안전하게 연결할 수 있다. 이를 위해 k8s는 po가 생성될 때 public root certificate와 유효한 bearer token을 자동으로 주입한다. po는 인증서가 아닌 bearer token을 이용해 kube-apiserver에 인증한다. default ns의 kubernetes svc는 kube-apiserver의 HTTPS endpoint로 redirect(kube-proxy가 수행)되는 virtual ip로 구성되어 있다.

control plane 구성 요소 역시 kube-apiserver와 안전한 포트를 사용해 통신한다.

결과적으로 no, po와 control plane의 연결은 기본적으로 안전하기 때문에 공용 네트워크에서 실행할 수 있다.

## Control plane to node
control plane(kube-apiserver)은 no와 통신하기 위해 두 가지 방식을 사용한다. 첫 번째는 kube-apiserver가 각 no의 kubelet과 통신한다. 두 번째는 kube-apiserver의 proxy 기능을 통해 모든 no, po, svc로 통신한다.

### API server to kubelet
kube-apiserver가 kubelet과 통신하는 이유는 다음과 같다.
- 필요에 따라 kube-apiserver가 kubelet에 no, po 상태에 대한 추가 정보 요청 등
- po의 로그조회 (`kubectl logs`)
- 실행 중인 po에 대한 attach(`kubectl exec`)
- kubelet의 port-forwarding 기능 제공(`kubectl port-forward`)

이러한 연결은 kubelet의 endpoint를 통해 이루어진다. 기본적으로 kube-apiserver는 kubelet의 서버 certificate를 확인하지 않으며 이로 인해 연결이 중간자(man-in-the-middle) 공격에 노출되어 신뢰할 수 없거나 공용 네트워크를 통해 실행하는 것이 안전하지 않다.

이러한 연결을 검증하기 위해 `--kubelet-certificate-authority` flag를 사용해 kube-apiserver에서 kubelet의 certificate를 검증할 수 있는 root certificate를 명시할 수 있다.

또는 kube-apiserver와 kubelet 사이에 [SSH tunneling](https://kubernetes.io/docs/concepts/architecture/control-plane-node-communication/#ssh-tunnels)을 사용할 수 있다.

또는 kubelet API를 보호하기 위해 [Kubelet authentication and/or authorization](https://kubernetes.io/docs/reference/access-authn-authz/kubelet-authn-authz/)을 참고한다.

### API server to nodes, pods, and services (apiserver proxy)
kube-apiserver에서 no, po, svc로의 연결은 기본적으로 일반 HTTP를 사용하며 안전하지 않다. API url에서 no, po, svc 이름 앞에 `https:`를 사용해 HTTPS 연결을 실행할 수 있지만 제공되는 certificate는 검증되지 않는다. 따라서 연결 자체는 암호화되지만 무결성에 대한 어떠한 보장도 제공하지는 않는다. 현재는 이러한 연결에 대해 신뢰할 수 없거나 공용 네트워크를 통해 실행하는 것은 안전하지 않다.

kube-apiserver의 proxy는 [Proxies in Kubernetes](https://kubernetes.io/docs/concepts/cluster-administration/proxies/) 페이지를 참고한다.

### SSH tunnels
k8s는 control plane과 no 간 통신 경로를 보호하기 위해 [SSH tunnel](https://www.ssh.com/academy/ssh/tunneling)을 지원한다. 이를 통해 kube-apiserver는 cluster의 각 no와 SSH tunnel을 초기화하며(listen port 22를 사용하는 ssh 서버) tunnel을 사용해 kubelet, no, svc로 향하는 모든 트래픽을 전달한다. 이 tunnel을 통해 트래픽이 no가 실행 중인 네트워크 외부에 노출되지 않도록 보장한다.

> **Note**:  
> SSH tunnel은 현재 deprecated 됐으며 사용하지 않는 것을 권장한다. 대신 Konnectivity service를 사용한다.

### Konnectivity service