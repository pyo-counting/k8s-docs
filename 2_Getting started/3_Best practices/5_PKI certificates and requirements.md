k8s는 내부적으로 유저 인증 정보를 저장하지 않는다. 대부분의 웹 서비스나 인증 서버들은 사용자 정보를 내부적으로 저장하여 사용자로부터 인증 정보를 전달 받았을 때 저장된 정보를 바탕으로 인증을 처리한(예를 들어 웹 사이트에서 계정과 비밀번호를 입력 받아 유저 DB를 조회하여 사용자 인증을 처리). k8s는 이와 다르게 따로 인증 정보를 저장하지 않고 외부 인증 시스템에서 제공해주는 신원 확인 기능들을 활용하여 사용자 인증을 하고 유저를 인식(identify)한다. 이러한 특징으로 인해 k8s에서는 쉽게 인증체계를 확장할 수 있다. k8s 내부 인증체계에 종속되는 부분이 거의 없기 때문이다. k8s는 사용자 인증체계를 전부 외부 시스템 (혹은 메커니즘)에 의존한다고 볼 수 있다. (X.509, HTTP Auth, Proxy Authentication 등)
- https://coffeewhale.com/kubernetes/authentication/x509/2020/05/02/auth01/

k8s는 TLS를 통한 인증을 위해 PKI(Public Key Infrastructure) certificate가 필요하다. kubeadm으로 k8s를 설치하면 cluster가 필요로 하는 certificate가 자동으로 생성된다. 또는 kube-apiserver에 key를 저장하지 않고 private key를 더 안전하게 보관하기 위해 직접 certificate를 생성 및 관리할 수도 있다. 이 페이지에서는 cluster에 필요한 certificate에 대해 설명한다.

## How certificates are used by your cluster
k8s는 아래 동작을 위해 PKI가 필요하다.

### Server certificates
- kube-apiserver가 제공하는 API를 위한 server certificate
- etcd server를 위한 server certificate
- kubelet을 위한 [server certificate](https://kubernetes.io/docs/reference/access-authn-authz/kubelet-tls-bootstrapping/#client-and-serving-certificates)
- (Optional) [front-proxy](https://kubernetes.io/docs/tasks/extend-kubernetes/configure-aggregation-layer/)를 위한 server certificate

### Client certificates
- kube-apiserver에 인증을 위해 kubelet이 사용하는 client certificate
- etcd에 인증을 위해 kube-apiserver가 사용하는 client certificate
- kube-apiserver와 안전하게 통신하기 위해 kube-controller-manager가 사용하는 client certificate
- kube-apiserver와 안전하게 통신하기 위해 kube-scheduler가 사용하는 client certificate
- kube-apiserver에 인증을 위해 kube-proxy가 사용하는 client certificate
- kube-apiserver에 인증을 위해 관리자가 사용하는 client certificate
- (Optional) [front-proxy](https://kubernetes.io/docs/tasks/extend-kubernetes/configure-aggregation-layer/)를 위한 client certificate

### Kubelet's server and client certificates
kube-apiserver가 kubelet과 안전하게 통신, 인증을 위해 client certificate와 key가 필요하다.

아래 두 가지 방식을 사용해 certificate를 이용할 수 있다.
- Shared Certificates: kube-apiserver는 client와 통신하기 위해 사용하는 server certificate를 동일하게 사용할 수 있다. 즉 `apiserver.cet`, `apiserver.key`를 kubelet과 통신하기 위해 client certificate로 사용할 수 있다.
- Separate Certificates: 위 방법 대신 kube-apiserver는 kubelet과 통신하기 위한 별도의 client certificate를 사용할 수 있다. 즉 `kubelet-client.crt`, `kubelet-client.key`를 대신 사용할 수 있다.

> **Note**:  
> front-proxy certificate는 [an Extension API server](https://kubernetes.io/docs/tasks/extend-kubernetes/setup-extension-api-server/)를 사용할 경우에만 필요하다.

etcd는 clinet, peer의 인증 목적을 위해 mutual TLS를 구현한다.

## Where certificates are stored
kubeadm을 사용해 k8s를 설치할 경우 대부분의 certificate는 `/etc/kubernetes/pki`에 저장된다. 해당 문서에서의 경로는 모두 해당 디렉토리의 상대경로다. 예외적으로 사용자를 위한 certificate는 `/etc/kubernetes`에 위치한다.

## Configure certificates manually
kubeadm이 모든 certificate를 만드는 대신 단일 root CA를 이용해 직접 생성하거나 필요한 모든 certificate를 직접 제공할 수 있다. CA 생성 방법은 [Certificates](https://kubernetes.io/docs/tasks/administer-cluster/certificates/)를 참고한다.

### Single root CA
관리자가 직접 관리할 수 있는 단일 root CA를 생성할 수 있다. 이 root CA를 이용해 intermediate CA를 생성한 후, 이후의 필요한 인증서 생성 작업은 k8s에 위임할 수 있다.

필요 CA 목록:
| path                   | Default CN                | description                    |
|------------------------|---------------------------|--------------------------------|
| ca.crt,key             | kubernetes-ca             | Kubernetes general CA          |
| etcd/ca.crt,key        | etcd-ca                   | For all etcd-related functions |
| front-proxy-ca.crt,key | kubernetes-front-proxy-ca | For the front-end proxy        |

> **Note**:  
> 왜그러는지 모르겠는데 대부분의 블로그에서 kubernetes general CA(ca.crt, ca.key)를 root CA라고 말한다. 아마도 k8s의 대부분의 구성요소에서 사용되는 CA라서 그런 것이 아닐까?

위 CA 이외에 sa 관리를 위한 public, private key가 필요하다: `sa.key`, `sp.pub`. 아래는 key, certificate의 경로를 나타낸다.
``` sh
/etc/kubernetes/pki/ca.crt
/etc/kubernetes/pki/ca.key
/etc/kubernetes/pki/etcd/ca.crt
/etc/kubernetes/pki/etcd/ca.key
/etc/kubernetes/pki/front-proxy-ca.crt
/etc/kubernetes/pki/front-proxy-ca.key
```

### All certificates
CA private key를 cluster에 복사하기 싫다면 직접 모든 certificate를 생성할 수 있다.

필요 certificate 목록:
|                               |                           |                |                |                                                              |
|-------------------------------|---------------------------|----------------|----------------|--------------------------------------------------------------|
| Default CN                    | Parent CA                 | O (in Subject) | kind           | hosts (SAN)                                                  |
| kube-etcd                     | etcd-ca                   |                | server, client | &lt;hostname&gt;, &lt;Host_IP&gt;, localhost, 127.0.0.1      |
| kube-etcd-peer                | etcd-ca                   |                | server, client | &lt;hostname&gt;, &lt;Host_IP&gt;, localhost, 127.0.0.1      |
| kube-etcd-healthcheck-client  | etcd-ca                   |                | client         |                                                              |
| kube-apiserver-etcd-client    | etcd-ca                   |                | client         |                                                              |
| kube-apiserver                | kubernetes-ca             |                | server         | &lt;hostname&gt;, &lt;Host_IP&gt;, &lt;advertise_IP&gt;, [1] |
| kube-apiserver-kubelet-client | kubernetes-ca             | system:masters | client         |                                                              |
| front-proxy-client            | kubernetes-front-proxy-ca |                | client         |                                                              |

> **Note**:  
> Instead of using the super-user group system:masters for kube-apiserver-kubelet-client a less privileged group can be used. kubeadm uses the kubeadm:cluster-admins group for that purpose.



### Certificate paths

``` sh
/etc/kubernetes/pki/etcd/ca.key
/etc/kubernetes/pki/etcd/ca.crt
/etc/kubernetes/pki/apiserver-etcd-client.key
/etc/kubernetes/pki/apiserver-etcd-client.crt
/etc/kubernetes/pki/ca.key
/etc/kubernetes/pki/ca.crt
/etc/kubernetes/pki/apiserver.key
/etc/kubernetes/pki/apiserver.crt
/etc/kubernetes/pki/apiserver-kubelet-client.key
/etc/kubernetes/pki/apiserver-kubelet-client.crt
/etc/kubernetes/pki/front-proxy-ca.key
/etc/kubernetes/pki/front-proxy-ca.crt
/etc/kubernetes/pki/front-proxy-client.key
/etc/kubernetes/pki/front-proxy-client.crt
/etc/kubernetes/pki/etcd/server.key
/etc/kubernetes/pki/etcd/server.crt
/etc/kubernetes/pki/etcd/peer.key
/etc/kubernetes/pki/etcd/peer.crt
/etc/kubernetes/pki/etcd/healthcheck-client.key
/etc/kubernetes/pki/etcd/healthcheck-client.crt
/etc/kubernetes/pki/sa.key
/etc/kubernetes/pki/sa.pub
```

## Configure certificates for user accounts
아래 관리자 계정은 반드시 직접 설정해야 한다.
| filename                | credential name            | Default CN                         | O (in Subject) |
|-------------------------|----------------------------|------------------------------------|----------------|
| admin.conf              | default-admin              | kubernetes-admin                   | \<admin-group> |
| super-admin.conf        | default-super-admin        | kubernetes-super-admin             | system:masters |
| kubelet.conf            | default-auth               | system:node:\<nodeName> (see note) | system:nodes   |
| controller-manager.conf | default-controller-manager | system:kube-controller-manager     |                |
| scheduler.conf          | default-scheduler          | system:kube-scheduler              |                |

> **Note**:  
> kubelet.conf 파일의 \<nodeName> 값은 kubelet이 kube-apiserver에 no를 등록할 때 제공하는 no 이름 값과 정확히 일치해야 한다. 상세 내용은 [Node Authorization](https://kubernetes.io/docs/reference/access-authn-authz/node/)을 참고한다.

> **Note**:  
> 위 예시에서 <admin-group>은 구현 방식에 따라 다릅니다. 일부 도구는 기본 admin.conf의 인증서를 system:masters 그룹에 속하도록 서명합니다. system:masters는 Kubernetes의 권한 부여 계층(RBAC 등)을 우회할 수 있는 일종의 긴급(super user) 그룹입니다. 또한 일부 도구는 이 슈퍼 유저 그룹에 바인딩된 인증서를 가진 별도의 super-admin.conf를 생성하지 않습니다.
>
> kubeadm은 두 관리자를 위한 인증서를 모두 생성한다. 첫 번째는 admin.conf kubeconfig 파일에서 사용되며 `Subject: O = kubeadm:cluster-admins, CN = kubernetes-admin` 값을 갖는다. ₩kubeadm:cluster-admins₩는 cluster-admin ClusterRole에 바인딩된 custom 그룹이다. 이 파일은 kubeadm으로 관리되는 모든 control plane 머신에서 생성된다.
>
> 두 번째는 super-admin.conf에서 사용되며 `Subject: O = system:masters, CN = kubernetes-super-admin` 값을 갖는다. 이 파일은 kubeadm init을 호출한 no에서만 생성된다.

1. 각 CN, O에 대해 x509 인증서/key 쌍을 생성 및 인증한다(`kubernetes-ca` 인증서로 인증).
2. 각 설정에 대해 아래 kubectl 명령어를 통해 kubeconfig 파일을 설정한다.
``` sh
KUBECONFIG=<filename> kubectl config set-cluster default-cluster --server=https://<host ip>:6443 --certificate-authority <path-to-kubernetes-ca> --embed-certs
KUBECONFIG=<filename> kubectl config set-credentials <credential-name> --client-key <path-to-key>.pem --client-certificate <path-to-cert>.pem --embed-certs
KUBECONFIG=<filename> kubectl config set-context default-system --cluster default-cluster --user <credential-name>
KUBECONFIG=<filename> kubectl config use-context default-system

```

파일들은 아래 용도로 사용된다.
| Filename                | Command                 | Comment                                                             |
|-------------------------|-------------------------|---------------------------------------------------------------------|
| admin.conf              | kubectl                 | Configures administrator user for the cluster                       |
| super-admin.conf        | kubectl                 | Configures super administrator user for the cluster                 |
| kubelet.conf            | kubelet                 | One required for each node in the cluster.                          |
| controller-manager.conf | kube-controller-manager | Must be added to manifest in manifests/kube-controller-manager.yaml |
| scheduler.conf          | kube-scheduler          | Must be added to manifest in manifests/kube-scheduler.yaml          |

아래는 실제 파일의 경로다.
``` sh
/etc/kubernetes/admin.conf
/etc/kubernetes/super-admin.conf
/etc/kubernetes/kubelet.conf
/etc/kubernetes/controller-manager.conf
/etc/kubernetes/scheduler.conf
```

1. any other IP or DNS name you contact your cluster on (as used by kubeadm the load balancer stable IP and/or DNS name, kubernetes, kubernetes.default, kubernetes.default.svc, kubernetes.default.svc.cluster, kubernetes.default.svc.cluster.local