k8s cluster에서 worker node의 구성 요소인 kubelet, kube-proxy는 kube-apiserver와 통신해야 한다. 이러한 통신이 안전하고 각 구성 요소와 다른 신뢰할 수 있는 구성 요소와의 통신을 보장하기 위해 no에 client certificate를 사용하는 것을 권장한다.

특히 kube-apiserver와 안전하게 통신하기 위해 certificate가 필요한 worker node와 같은 구성 요소를 bootstrap하는 일반적인 프로세스는 종종 k8s의 범위를 벗어나고 중요한 추가 작업이 필요하기 때문에 어려울 수 있다. 이로 인해 cluster를 초기화하거나 확장하는 것이 어려워질 수 있다.

이 프로세스를 간소화하기 위해 k8s 버전 1.4에서 certificate request, signing API를 도입했다. 제안 내용은 [here](https://github.com/kubernetes/kubernetes/pull/20439)에서 확인할 수 있다.

이 문서는 no의 initialization, kubelet을 위한 TLS client certificate bootstrap 설정, 동작 방식에 대해 설명한다.

## Initialization process
worker node 내에서 kubelet은 아래 동작을 수행한다.
1. `kubeconfig` 파일을 확인한다.
2. `kubeconfig` 파일 내에서 kube-apiserver의 URL, credential(TLS key, 서명된 certificate)을 찾는다.
3. credential을 사용해 kube-apiserver에 통신을 시도한다.

kube-apiserver가 kubelet의 credential을 유효성을 확인하면 kubelet을 유효한 no로 인지하고 po를 할당하기 시작한다.

위 과정은 아래 내용이 전제되어야 한다.
- worker node의 `kubeconfig` 파일 내 key, certificate가 있어야 한다.
- certificate는 kube-apiserver가 신뢰하는 CA에 의해 서명되어야 한다.

이를 위해 관리자는 아래 작업을 선행해야 한다.
1. CA key, certificate를 생성한다.
2. kube-apiserver가 실행되는 control plane no에 CA certificate를 배포한다.
3. 각 kubelet을 위한 key, certificate를 생성한다. 각 kubelet마다 고유한 CN을 갖는 것을 권장한다.
4. CA key를 사용해 kubelet certificat를 서명한다.
5. kubelet이 실행되는 각 no에 kubelet key와 서명한 certificate를 배포한다.

이 문서에서 설명하는 TLS bootstrap은 cluster를 초기화하거나 확장할 때 3번 이후의 과정을 k8s가 어떻게 단순화하고 자동화하는지 설명한다.

### Bootstrap initialization
bootstrap initialization 프로세스에서 다음과 같은 동작이 발생한다.
1. kubelet이 실행된다.
2. kubelet이 `kubeconfig` 파일이 없음을 확인한다.
3. kubelet이 `bootstrap-kubeconfig` 파일을 확인한다.
4. kubelet은 `bootstrap-kubeconfig` 파일에서 kube-apiserver의 URL과 사용 제한이 있는 "token"을 발견한다.
5. kubelet은 token을 사용해 kube-apiserver에 인증한다.
6. kubelet은 csr을 생성하고 검색할 수 있는 제한된 credential을 갖게된다.
7. kubelet은 signerName을 `kubernetes.io/kube-apiserver-client-kubelet`로 설정한 scr를 생성한다.
8. csr은 아래 중 한 가지 방법을 통해 승인된다.
    - 설정된 경우, kube-controller-manager가 csr을 자동 승인한다.
    - 설정된 경우, k8s API, kubectl을 사용해 scr을 수동 승인한다.
9. kubelet을 위한 certificate가 생성된다.
10. kubelet을 위한 certificate가 발행된다.
11. kubelet은 certificate를 검색한다.
12. kubelet은 key, 서명된 certificate를 사용해 적절한 `kubeconfig`를 생성한다.
13. kubelet은 정상 동작을 수행한다.
14. 옵션: 설정된 경우, certificate가 만료도기 진에 kubelet이 자동으로 갱신 요청을 수행한다.
15. 갱신 certificate은 설정에 따라 자동 또는 수동으로 생성, 발급된다.

아래에서는 TLS bootstrapping을 위한 필요 설정, 제한 사항을 설명한다.

## Configuration

## Certificate Authority

### kube-apiserver configuration