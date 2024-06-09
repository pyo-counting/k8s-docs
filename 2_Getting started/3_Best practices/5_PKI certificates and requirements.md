k8s는 내부적으로 유저 인증 정보를 저장하지 않는다. 대부분의 웹 서비스나 인증 서버들은 사용자 정보를 내부적으로 저장하여 사용자로부터 인증 정보를 전달 받았을 때 저장된 정보를 바탕으로 인증을 처리한(예를 들어 웹 사이트에서 계정과 비밀번호를 입력 받아 유저DB를 조회하여 사용자 인증을 처리). k8s는 이와 다르게 따로 인증 정보를 저장하지 않고 각각의 인증 시스템에서 제공해주는 신원 확인 기능들을 활용하여 사용자 인증을 하고 유저를 인식(identify)한다. 이러한 특징으로 인해 k8s에서는 쉽게 인증체계를 확장할 수 있다. k8s 내부 인증체계에 종속되는 부분이 거의 없기 때문이다. k8s는 사용자 인증체계를 전부 외부 시스템 (혹은 메커니즘)에 의존한다고 볼 수 있다. (X.509, HTTP Auth, Proxy Authentication 등)
- https://coffeewhale.com/kubernetes/authentication/x509/2020/05/02/auth01/

k8s는 TLS를 통한 인증을 위해 PKI(Public Key Infrastructure) certificate가 필요하다. kubeadm으로 k8s를 설치하면 cluster가 필요로 하는 certificate가 자동으로 생성된다. 또는 kube-apiserver에 key를 저장하지 않고 private key를 더 안전하게 보관하기 위해 직접 certificate를 생성 및 관리할 수도 있다. 이 페이지에서는 cluster에 필요한 certificate에 대해 설명한다.

## How certificates are used by your cluster
k8s는 아래 동작을 위해 PKI가 필요하다.
- kubelet이 kube-apiserver에 인증하기 위한 client certificate
- kube-apiserver가 kubelet에 인증하기 위한 kubelet server certificate
- kube-apiserver의 endpoint를 위한 server certificate
- cluster의 관리자가 kube-apiserver에 인증하기 위한 client certificate
- kube-apiserver가 kubelet에 인증하기 위한 client certificate
- kube-apiserver가 etcd에 인증하기 위한 client certificate
- Client certificate/kubeconfig for the controller manager to talk to the API server
- Client certificate/kubeconfig for the scheduler to talk to the API server.
- [front-proxy](https://kubernetes.io/docs/tasks/extend-kubernetes/configure-aggregation-layer/)를 위한 client, server certificate

> **Note**:  
> front-proxy certificate는 [an Extension API server](https://kubernetes.io/docs/tasks/extend-kubernetes/setup-extension-api-server/)를 사용할 경우에만 필요하다.

## Where certificates are stored
kubeadm을 사용해 k8s를 설치할 경우 대부분의 certificate는 `/etc/kubernetes/pki`에 저장된다. 해당 문서에서의 경로는 모두 해당 디렉토리의 상대경로다. 예외적으로 사용자를 위한 certificate는 `/etc/kubernetes`에 위치한다.

## Configure certificates manually
kubeadm이 모든 certificate를 만드는 대신 직접 root CA를 생성하거나 필요한 모든 certificate를 직접 제공할 수 있다. CA 생성 방법은 [Certificates](https://kubernetes.io/docs/tasks/administer-cluster/certificates/)를 참고한다.

### Single root CA
관리자가 관리하는 단일 root CA를 생성할 수 있다. root CA는 intermediate CA를 생성할 수 있고 이후의 작업은 k8s에 위임할 수 있다.

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
아래 관리자 계정, sa를 반드시 생성해야 한다.
| filename                | credential name            | Default CN                         | O (in Subject) |
|-------------------------|----------------------------|------------------------------------|----------------|
| admin.conf              | default-admin              | kubernetes-admin                   | \<admin-group> |
| super-admin.conf        | default-super-admin        | kubernetes-super-admin             | system:masters |
| kubelet.conf            | default-auth               | system:node:\<nodeName> (see note) | system:nodes   |
| controller-manager.conf | default-controller-manager | system:kube-controller-manager     |                |
| scheduler.conf          | default-scheduler          | system:kube-scheduler              |                |
