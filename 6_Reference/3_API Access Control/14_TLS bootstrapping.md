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
3. 각 kubelet을 위한 key, certificate를 생성한다. 각 kubelet 마다 고유한 CN을 갖는 것을 권장한다.
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
TLS bootstrapping, (optional) 자동 승인을 설정하기 위해 아래 구성 요소를 설정해야 한다.
- kube-apiserver
- kube-controller-manager
- kubelet
- `ClusterRoleBinding`, `ClusterRole` resource

추가적으로 k8s ca도 필요하다.

## Certificate Authority
bootstrapping의 유무와 상관없이 ca key, 인증서는 필요하며 이는 kubelet 인증서를 서명하는 데 사용한다. 이 파일은 사용자가 k8s control plane에 배포해야 한다.

이 문서에서는 인증서가 control plane의 `/var/lib/kubernetes/ca.pem`(인증서), `/var/lib/kubernetes/ca-key.pem`(key) 경로에 배포되었다고 가정한다. 이러한 정보를 "k8s ca 인증서, key"라고 부른다.

이러한 인증서를 사용하는 모든 k8s 구성 요소(kubelet, kube-apiserver, kube-controller-manager)는 key와 인증서를 PEM으로 인코딩된 것으로 가정한다.

## kube-apiserver configuration
TLS bootstrapping을 활성화 하기 위해 kube-apiserver에 대한 몇 가지 요구 사항이 있다.
- 클라이언트 인증서를 서명하는 CA 정보에 대한 인식
- bootstrapping kubelet을 `system:bootstrappers` group에 인증
- bootstrapping kubelet이 인증서 서명 요청(scr)을 할 수 있도록 인가

### Recognizing client certificates
이는 모든 클라이언트 인증서 인증에 공통 사항이다. 클라이언트 인증서 인증을 활성화하기 위해 kube-apiserver에 `--client-ca-file=FILENAME` flag를 추가하고, 서명 인증서를 포함하는 sa bundle을 참조하는 파일 경로를 참조한다. 예를 들어 `--client-ca-file=/var/lib/kubernetes/ca.pem`와 같이 지정할 수 있습다.

### Initial bootstrap authentication
bootstrapping kubelet이 kube-apiserver에 연결해 인증서를 요청하기 위해 먼저 서버에 인증해야 한다. kubelet을 인증할 수 있는 모든 authenticator을 사용할 수 있다.

kubelet의 초기 bootstrap credential에는 어떤 authentication 방법을 사용해도 상관 없지만 다음 두 가지 authenticator를 사용하는 것이 쉽다.
1. bootstrap tokens
2. static token file

부트스트랩 토큰을 사용하면 kubelet을 인증하는 것이 더 간단하고 관리하기 쉬우며, kube-apiserver를 시작할 때 추가 플래그가 필요하지 않습니다.

어느 방법을 선택하든, 요구 사항은 다음과 같습니다. kubelet은 다음 권한을 가진 사용자로 인증할 수 있어야 합니다.

- CSR을 생성하고 검색할 수 있는 권한
- 자동 승인이 활성화된 경우 노드 클라이언트 인증서를 요청하면 자동으로 승인될 수 있는 권한
부트스트랩 토큰을 사용하여 인증하는 kubelet은 시스템:부트스트래퍼스 그룹의 사용자로 인증되며, 이는 일반적으로 사용되는 방법입니다.

이 기능이 성숙해짐에 따라 토큰이 인증서 프로비저닝과 관련된 클라이언트 요청에 엄격하게 제한되는 역할 기반 액세스 제어 (RBAC) 정책에 바인딩되어 있는지 확인해야 합니다. RBAC가 적용된 상태에서 토큰을 그룹에 스코핑하면 매우 유연하게 사용할 수 있습니다. 예를 들어, 노드 프로비저닝이 완료되면 특정 부트스트랩 그룹의 액세스를 비활성화할 수 있습니다.
#### Bootstrap tokens

#### Token authentication file

### Authorize kubelet to create CSR

## kube-controller-manager configuration
### Access to key and certificate
### Approval
## kubelet configuration
### Client and serving certificates
### Certificate rotation
## Other authenticating components
## kubectl approval