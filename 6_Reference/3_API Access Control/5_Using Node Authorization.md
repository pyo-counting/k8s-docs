node authorization은 kubelet의 API 요청을 인가하기 위한 특수한 목적의 authorization이다.

## Overview
node authorizor는 아래 kubelet API 동작을 허용한다.

- read
    - services
    - endpoints
    - nodes
    - pods
    - secrets, configmaps, persistent volume claims and persistent volumes related to pods bound to the kubelet's node
- write
    - nodes and node status (enable the NodeRestriction admission plugin to limit a kubelet to modify its own node)
    - pods and pod status (enable the NodeRestriction admission plugin to limit a kubelet to modify pods bound to itself)
    - events
- 인증 관련
    - read/write access to the CertificateSigningRequests API for TLS bootstrapping
    - the ability to create TokenReviews and SubjectAccessReviews for delegated authentication/authorization checks

향후 릴리스에서 node authorizor가 kubelet이 올바르게 동작하기 위해 필요한 최소한의 권한만 갖도록 권한을 추가하거나 제거할 수 있다.

node authorizor에 의해 인증되기 위해 kubelet은 system `system:node:<nodeName>` 사용자, `system:nodes` 그룹에 대한 credentials을 사용해야 한다. 해당 사용자, 그룹 형식은 [kubelet TLS bootstrapping](https://kubernetes.io/docs/reference/access-authn-authz/kubelet-tls-bootstrapping/)을 통해 생성된 kubelet의 identity와 일치한다.

`<nodeName>` 값은 kubelet이 등록한 no 이름과 정확히 일치해야 한다. 기본적으로 이는 hostname에 의해 제공되는 호스트 이름이거나 kubelet의 `--hostname-override` flag를 통해 제공된 값이다. 그러나 kubelet의 `--cloud-provider` flag를 사용하는 경우 hostname과 `--hostname-override` flag를 무시하고 cloud provider에 의해 결정될 수 있다. kubelet이 hostname을 결정하는 방법에 대한 자세한 내용은 [kubelet options reference](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/)를 참고한다.

node authorizor는 kube-apiserver의 `--authorization-mode=Node` flag를 사용해 활성화한다.

kubelet이 write할 수 있는 API object를 제한하기 위헤 kube-apiserver의 `--enable-admission-plugins=...,NodeRestriction,...` flag를 사용해 [NodeRestriction](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers#noderestriction) admission controller를 활성화한다.

## Migration considerations
### Kubelets outside the `system:nodes` group
`system:nodes` 그룹에 속하지 않는 kubelet은 node authorization에 의해 인가되지 않는다. node admission plugin은 이러한 kubelet의 요청에 대해서는 제한하지 않는다.

### Kubelets with undifferentiated usernames
배포판에 따라 kubelet이 `system:nodes` 그룹에 속하지만 `system:node:...` 포맷이 아닌 사용자여서 특정 no로 식별되지 않는다. 이러한 kubelet은 node authorizor에 의해 인가되지 않기 때문에 다른 메커니즘에 의해 인가되어야 한다.

`NodeRestriction` admission plugin은 이러한 kubelet의 요청을 무시한다. 왜냐하면 기본 node identifier에 대한 구현이 이를 node identity로 간주하지 않기 때문이다.