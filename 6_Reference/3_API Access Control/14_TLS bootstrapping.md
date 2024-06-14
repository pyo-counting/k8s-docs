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
4. kubelet은 `bootstrap-kubeconfig` 파일에서 kube-apiserver의 URL과 사용 제한이 있는 token(bootstrap token)을 발견한다.
    - 해당 파일에는 kube-apiserver의 ca 정보도 포함할 수 있다. ca에 대한 정보가 파일에 없다면 kube-public ns의 cluster-info cm에서 정보를 가져오는 단계도 있을 것 같다. 정확하게 해당 프로세스가 어떻게 수행되는지는 kubeadm을 살펴봐야한다.
5. kubelet은 token을 사용해 kube-apiserver에 인증한다.
6. kubelet은 csr을 생성하고 검색할 수 있는 제한된 credential을 갖게된다.
7. kubelet은 `.spec.signerName` 필드를 `kubernetes.io/kube-apiserver-client-kubelet`로 설정한 csr object를 생성한다.
8. csr은 아래 중 한 가지 방법을 통해 승인된다.
    - 설정된 경우, kube-controller-manager가 csr을 자동 승인한다.
    - 설정된 경우, k8s API, kubectl을 사용해 csr을 수동 승인한다.
9. kubelet을 위한 certificate가 생성된다.
10. kubelet을 위한 certificate가 발행된다.
11. kubelet은 certificate를 검색한다.
12. kubelet은 key, 서명된 certificate를 사용해 적절한 `kubeconfig`를 생성한다.
13. kubelet은 정상 동작을 수행한다.
14. 옵션: 설정된 경우, certificate가 만료되기 진에 kubelet이 자동으로 갱신 요청을 수행한다.
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
- bootstrapping kubelet이 인증서 서명 요청(csr)을 할 수 있도록 인가

### Recognizing client certificates
이는 모든 클라이언트 인증서 인증에 공통 사항이다. 클라이언트 인증서 인증을 활성화하기 위해 kube-apiserver에 `--client-ca-file=FILENAME` flag를 추가하고, 서명 인증서를 포함하는 sa bundle을 참조하는 파일 경로를 참조한다. 예를 들어 `--client-ca-file=/var/lib/kubernetes/ca.pem`와 같이 지정할 수 있습다.

### Initial bootstrap authentication
bootstrapping kubelet이 kube-apiserver에 연결해 인증서를 요청하기 위해 먼저 서버에 인증해야 한다. kubelet을 인증할 수 있는 모든 authenticator을 사용할 수 있다.

kubelet의 초기 bootstrap credential에는 어떤 authentication 방법을 사용해도 상관 없지만 다음 두 가지 authenticator를 사용하는 것이 쉽다.
1. bootstrap token
2. static token file

bootstrap token을 사용하면 kubelet을 인증하는 것이 더 간단하고 관리하기 쉬우며, kube-apiservr에 추가 flag가 필요하지 않다.

authentication의 종류와 상관없이 요구 사항은 다음과 같다. kubelet은 다음 권한을 가진 사용자로 인증할 수 있어야 한다.
- csr을 생성, 검색할 수 있는 권한
- 자동 승인이 활성화된 경우, no의 client 인증서를 요청하면 자동으로 승인될 수 있는 권한

bootstrap token을 사용해 인증하는 kubelet은 `system:bootstrappers` 그룹의 사용자로 인증된다.

이 기능이 성숙해짐에 따라 token이 인증서 provisioning과 관련된 클라이언트 요청으로 엄격하게 제한되는 RBAC 정책에 바인딩되어 있는지 확인해야 한다. RBAC가 적용된 상태에서 token을 그룹에 매핑하면 매우 유연하게 사용할 수 있다. 예를 들어, no provisioning이 완료되면 특정 bootstrap 그룹의 접근을 비활성화할 수 있다.

#### Bootstrap tokens
bootstrap token은 k8s cluster에 secret로 저장되고 개별 kubelet에게 발급된다. 하나의 token을 전체 cluster에 사용하거나 no 마다 하나씩 발급할 수 있다.

이 과정은 두 가지로 나뉜다.
1. `bootstrap.kubernetes.io/token` type의 secret을 생성한다.
2. kubelet에 token을 발급한다.

kubelet 관점에서 하나의 token은 다른 것과 마찬가지로 특별한 의미가 없다. 그러나 kube-apiserver의 관점에서는 bootstrap token이 특별하다. type, ns, 이름으로 kube-apiserver는 특별한 token으로 인식하고 token을 사용해 인증하는 사용자에게 특별한 bootstrap 권한을 부여한다. 사용자는 `system:bootstrappers`에 속한다. 이는 TLS bootstrapping의 기본 요구 사항을 만족한다.

secret 생성 상세 과정은 [here](https://kubernetes.io/docs/reference/access-authn-authz/bootstrap-tokens/)을 참고한다.

bootstrap token을 사용하기 위해 kube-apiserver에 아래 flag를 사용한다.
``` sh
--enable-bootstrap-token-auth=true
```

#### Token authentication file
kube-apiserver은 인증을 위해 token을 사용할 수 있다. 이러한 token은 임의의 값이지만 적어도 안전한 난수 생성기(예를 들어 대부분의 최신 리눅스 시스템에서의 /dev/urandom)에서 파생된 적어도 128비트의 엔트로피를 나타내어야 한다. token을 생성하는 여러 방법이 있다. 아래는 예시다.
``` sh
head -c 16 /dev/urandom | od -An -t x | tr -d ' '
```

위는 `02b50b05283e98dd0fd71db496ef01e8`와 같은 형태의 token을 생성한다.

token 파일은 아래 예시와 같다.
```
02b50b05283e98dd0fd71db496ef01e8,kubelet-bootstrap,10001,"system:bootstrappers"
```

token 파일을 활성화하기 위해 kube-apiserver에 `--token-auth-file=FILENAME`를 추가한다.

### Authorize kubelet to create CSR
bootstrapping no가 `system:bootstrappers` 그룹의 일부로 인증되면, csr을 생성하고, 완료되면 검색할 수 있도록 허가되어야 한다. k8s는 이러한 권한을 갖는 `system:node-bootstrapper` ClusterRole을 제공한다.

이를 위해 `system:bootstrappers` 그룹을 cluster role `system:node-bootstrapper`에 바인딩하는 ClusterRoleBindings을 생성한다.
``` yaml
# enable bootstrapping nodes to create CSR
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: create-csrs-for-bootstrapping
subjects:
- kind: Group
  name: system:bootstrappers
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: system:node-bootstrapper
  apiGroup: rbac.authorization.k8s.io
```

## kube-controller-manager configuration
kube-apiserver가 kubelet으로부터 csr을 요청받고 해당 요청을 인증하는 반면, kube-controller-manager는 실제 서명된 인증서를 발급하는 역할을 수행한다.

kube-controlelr-manager는 certificate-issuing control loop을 통해 이 작업을 수행한다. 이는 디스크의 asset을 사용하는 [CFSSL](https://blog.cloudflare.com/introducing-cfssl/) local signer 형태를 갖는다. 현재 발급된 모든 인증서는 1년 동안 유효하며 기본 키 사용 용도로 설정된다.

kube-controller-manager가 인증서를 서명하기 위해 아래 요구 사항을 만족해야한다.
- "k8s CA key, 인증서"에 대한 접근
- csr signing 활성화

### Access to key and certificate
k8s ca key, 인증서를 생성하고 control plane에 배포해야 한다. 이는 kube-controller-manager가 kubelet 인증서를 서명하기 위해 사용된다.

서명된 인증서는 다시 kubelet이 kube-apiserver에 인증하기 위해 사용되기 때문에 kube-controller-manager에 제공되는 ca로 kube-apiserver에 의해 신뢰되어야 한다. 이는 kube-apiserver의 `--client-ca-file=FILENAME` flag를 통해 설정된다(예를 들어 `--client-ca-file=/var/lib/kubernetes/ca.pem`).

kube-controller-manager에 대한 flag 설정은 다음과 같다.
``` sh
--cluster-signing-cert-file="/etc/path/to/kubernetes/ca/ca.crt" --cluster-signing-key-file="/etc/path/to/kubernetes/ca/ca.key"
```

아래는 예시다.
``` sh
--cluster-signing-cert-file="/var/lib/kubernetes/ca.pem" --cluster-signing-key-file="/var/lib/kubernetes/ca-key.pem"
```

서명된 인증서의 유효 기간은 아래 flag를 통해 설정할 수 있다.
``` sh
--cluster-signing-duration
```

### Approval
csr을 승인하기 위해 kube-controller-manager에게 해당 승인이 허용되는 것이라고 알려줘야 한다. 이를 위해 올바른 group에 RBAC 권한을 부여해야 한다. 이를 통해 kube-controller-manager의 내장 signer는 자동 승인을 수행한다.

두 가지 구분된 권한 집합이 있다.
- `nodeclient`: no를 위해 새로운 인증서를 생성하는 경우 아직 인증서가 존재하지 않는다. 위에서 설명한 token 중 하나를 사용해 인증되므로 `system:bootstrappers` 그룹에 속하게된다.
- `selfnodeclient`: no가 인증서를 갱신하는 경우 이미 인증서가 존재한다. 그렇기 때문에 `system:nodes` 그룹으로 인증된다.

kubelet이 새 인증서를 요청하고 수신하기 위해 bootstrapping no가 속한 `system:bootstrappers` 그룹을 `system:certificates.k8s.io:certificatesigningrequests:nodeclient` ClusterRole과 바인딩하기 위한 ClusterRoleBinding을 생성한다.
``` yaml
# Approve all CSRs for the group "system:bootstrappers"
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: auto-approve-csrs-for-group
subjects:
- kind: Group
  name: system:bootstrappers
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: system:certificates.k8s.io:certificatesigningrequests:nodeclient
  apiGroup: rbac.authorization.k8s.io
```

kubelet이 인증서를 갱신할 수 있도록 no가 속한 `system:nodes` 그룹을 `system:certificates.k8s.io:certificatesigningrequests:selfnodeclient` ClusterRole과 바인딩하기 위한 ClusterRoleBindings을 생성한다.
``` yaml
# Approve renewal CSRs for the group "system:nodes"
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: auto-approve-renewals-for-nodes
subjects:
- kind: Group
  name: system:nodes
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: system:certificates.k8s.io:certificatesigningrequests:selfnodeclient
  apiGroup: rbac.authorization.k8s.io
```

kube-controller-manager의 일부로 제공되는 기본 활성화된 `csrapproving` controller가 있다. 이 controller는 주어진 사용자가 csr을 요청할 권한이 있는지 여부를 결정하기 위해 SubjectAccessReview API를 사용하고 승인을 수행해야 한다. 다른 signer와의 충돌을 방지하기 위해 내장된 signer는 csr을 명시적으로 거부하지 않는다. 인가되지 않은 요청만 무시된다. 또한 controller는 gc의 일환으로 만료된 인증서를 정리한다.

## kubelet configuration
kubelet을 bootstrap을 수행하기 위해 다음 요구사항이 필요하다.
- (optional) bootstrap을 통해 생성되는 key와 인증서 저장 경로
- 아직 존재하지 않는 kubeconfig 파일의 경로. bootstrap 이후 설정 파일의 경로
- 서버의 URL, bootstrap credential을 제공하기 위한 bootstrap kubeconfig 파일의 경로
- (optional) 인증서를 rotation하기 위한 절차

bootstrap kubeconfig는 kubelet이 접근할 수 있는 경로에 있어야 한다(예를 들어 `/var/lib/kubelet/bootstrap-kubeconfig`)

이 파일의 형식은 일반 kubeconfig 파일과 동일하다. 아래는 예시다.
``` yaml
apiVersion: v1
kind: Config
clusters:
- cluster:
    certificate-authority: /var/lib/kubernetes/ca.pem
    server: https://my.server.example.com:6443
  name: bootstrap
contexts:
- context:
    cluster: bootstrap
    user: kubelet-bootstrap
  name: bootstrap
current-context: bootstrap
preferences: {}
users:
- name: kubelet-bootstrap
  user:
    token: 07401b.f395accd246ae52d
```

중요 필드는 다음과 같다.
- `.clusters.[*].cluster.certificate-authority`: kube-apiserver server를 검증하기 위한 CA 파일 경로
- `.clusters.[*].cluster.server`: kube-apiserver의 URL
- `.users.[*].token`: 사용할 token

token의 형식은 kube-apiserver가 예상하는 형식과 일치하면 되므로 형식은 중요하지 않다. 위 예제에서는 bootstrap token을 사용했다. 다른 인증 방법을 사용해도 상관없다.

bootstrap kubeconfig는 표준 kubeconfig이므로 kubectl을 사용하여 생성할 수 있다. 위 예제 파일을 생성하기 위한 명령어는 다음과 같다.
``` sh
kubectl config --kubeconfig=/var/lib/kubelet/bootstrap-kubeconfig set-cluster bootstrap --server='https://my.server.example.com:6443' --certificate-authority=/var/lib/kubernetes/ca.pem
kubectl config --kubeconfig=/var/lib/kubelet/bootstrap-kubeconfig set-credentials kubelet-bootstrap --token=07401b.f395accd246ae52d
kubectl config --kubeconfig=/var/lib/kubelet/bootstrap-kubeconfig set-context bootstrap --user=kubelet-bootstrap --cluster=bootstrap
kubectl config --kubeconfig=/var/lib/kubelet/bootstrap-kubeconfig use-context bootstrap
```

kubelet이 bootstrap kubeconfig를 사용하기 위해 아래 flag를 사용한다.
``` sh
--bootstrap-kubeconfig="/var/lib/kubelet/bootstrap-kubeconfig" --kubeconfig="/var/lib/kubelet/kubeconfig"
```

kubelet을 시작할 때, `--kubeconfig`로 지정된 파일이 존재하지 않으면 `--bootstrap-kubeconfig`로 지정된 bootstrap kubeconfig가 사용되어 kube-apiserver에 클라이언트 인증서를 요청한다. 인증서 요청이 승인되고 kubelet이 인증서를 받으면 `--kubeconfig`로 지정된 경로에 생성된 키와 인증서를 참조하는 kubeconfig 파일이 작성된다. 인증서와 키 파일은 `--cert-dir`로 지정된 디렉터리에 위치한다.

### Client and serving certificates
위 내용은 모두 kubelet 클라이언트 인증서와 관련이 있다. 즉, kubelet이 kube-apiserver에 인증하는 데 사용하는 인증서다.

뿐만 아니라 kubelet은 serving 인증서를 사용할 수 있다. kubelet 자체는 특정 기능을 위해 https endpoint를 노출한다. endpoint를 안전하게 하기 kubelet은 아래를 수행할 수 있다.
- `--tls-private-key-file`, `--tls-cert-file` flag를 통해 제공된 키와 인증서를 사용한다.
- 키와 인증서가 제공되지 않은 경우, 자체 서명된 키와 인증서를 생성한다.
- CSR API를 통해 cluster 서버로부터 serving 인증서를 요청한다.

TLS bootstrapping에 의해 제공된 클라이언트 인증서는 기본적으로 client auth에만 사용되도록 서명되므로 serving 인증서 또는 server auth에 사용할 수 없다.

그러나 인증서 rotation을 통해 적어도 부분적으로 서버 인증서를 활성화할 수 있다.

### Certificate rotation
k8s v1.8 이상의 kubelet은 클라이언트, serving 인증서의 rotation 기능을 구현한다. 참고로, serving 인증서의 rotation은 베타 기능이며 kubelet에 RotateKubeletServerCertificate feature flag가 있어야 한다.(kubelet에서 기본적으로 활성화됨).

존재하는 credential이 만료될 때 새로운 csr을 생성해 클라이언트 인증서를 rotation할 수 있도록 kubelet을 설정할 수 있다. 이 기능을 활성화하려면 kubelet 설정 파일의 rotateCertificates 필드를 사용하거나 kubelet에 다름 flag를 설정한다(deprecated).
``` sh
--rotate-certificates
```

RotateKubeletServerCertificate를 활성화하면 kubelet은 클라이언트 credential을 bootstrapping한 후 serving 인증서를 요청하고 해당 인증서를 rotation한다. 이 동작을 활성화하려면 kubelet 설정 파일의 serverTLSBootstrap 필드를 사용하거나 kubelet에 다름 flag를 설정한다(deprecated).
``` sh
--rotate-server-certificates
```

> **Note**:  
> k8s core에 구현된 csr 승인 controller는 보안상의 이유로 node serving 인증서를 승인하지 않는다. RotateKubeletServerCertificate를 사용하기 위해 관리자는 custom 승인 controller를 실행하거나 serving 인증서 요청을 수동으로 승인해야 한다.
>
> kubelet serving 인증서에 대한 배포별 승인 프로세스는 일반적으로 다음과 같은 csr만 승인해야 한다.
> 1. are requested by nodes (ensure the spec.username field is of the form system:node:<nodeName> and spec.groups contains system:nodes)
> 2. request usages for a serving certificate (ensure spec.usages contains server auth, optionally contains digital signature and key encipherment, and contains no other usages)
> 3. only have IP and DNS subjectAltNames that belong to the requesting node, and have no URI and Email subjectAltNames (parse the x509 Certificate Signing Request in spec.request to verify subjectAltNames)

## Other authenticating components
이 문서에서 설명한 모든 TLS bootstrapping은 kubelet과 관련이 있다. 그러나 다른 구성 요소도 직접적으로 kube-apiserver와 통신해야 할 수 있다. 특히 중요한 것은 kube-proxy다. kube-proxy는 k8s no 구성 요소의 일부이며 모든 no에서 실행되지만 모니터링 또는 네트워킹과 같은 다른 구성 요소도 포함될 수 있다.

kubelet과 마찬가지로 이러한 다른 구성 요소도 kube-apiserver에 인증하는 방법이 필요하다. 이러한 자격 증명을 생성하는 여러 가지 옵션이 있다.
- 과거 방식: TLS bootstrapping 이전에 kubelet과 동일한 방식으로 인증서를 생성하고 배포한다.
- DaemonSet: kubelet 자체가 각 no에 로드되며 기본 서비스를 시작하는 데 충분하므로, kube-proxy, 다른 no 별 서비스를 독립 프로세스로 실행하는 대신 kube-system ns에서 ds로 실행할 수 있다. cluster 내부에 있으므로 적절한 sa를 제공해 활동을 수행할 적절한 권한을 부여할 수 있다. 이는 이러한 서비스를 구성하는 가장 간단한 방법이다.

## kubectl approval
csr은 kube-controller-manager에 내장된 승인 플로우가 아닌 외부에서 승인할 수 있다.

서명 controller는 모든 인증서 요청을 즉시 서명하지 않는다. 대신 해당 요청이 적절한 권한을 가진 사용자에 의해 "approved" 상태로 표시될 때까지 기다린다. 이 플로우는 외부 승인 controller 또는 kube-controller-manager에 구현된 승인 컨트롤러에 의해 자동 승인이 가능하도록 의도됐다. 그러나 cluster 관리자는 kubectl을 사용하여 수동으로 인증서 요청을 승인할 수도 있다. 관리자는 `kubectl get csr`을 사용하여 csr을 나열하고 `kubectl decsribe csr <name>`을 사용하여 자세히 볼 수 있다. 관리자는 `kubectl certificate approve <name>`, `kubectl certificate deny <name>`을 사용하해 csr을 승인하거나 거부할 수 있다.