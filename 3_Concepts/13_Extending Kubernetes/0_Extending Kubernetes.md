k8s는 확장성이 좋다. 그렇기 때문에 사용자가 k8s 프로젝트 코드의 fork, submit patch를 할 필요가 거의 없다.

이 가이드는 k8s cluster의 customization의 옵션에 대해 설명한다. 사용자의 요구 사항에 맞도록 k8s cluster를 조정하는 방법을 이해하기 원하는 운영자에게 도움이 될 수 있다. 플랫폼 개발자, k8s 프로젝트 contributor는 extension point, pattern에 대해 확인할 수 있다.

customization은 크게 CLI argument, 로컬 설정 파일, API resource에 대한 변경을 포함하는 configuration과 추가 프로그램 실행, 네트워크 서비스 실행을 포함하는 extension이 있다. 여기서는 주로 extension에 대해 설명한다.

## Configuration
설정 파일, CLI argument는 각 구성요소의 documentation을 확인한다.
- kube-apiserver
- kube-controller-manager
- kube-scheduler
- kubelet
- kube-proxy

CLI argument, 설정 파일은 항상 변경할 수 있는 것은 아니다. 변경이 가능하더라도 일반적으로 이는 cluster 운영자만 변경할 수 있다. 또한 이러한 변경 사항은 향후 k8s 버전에서 변경될 수 있으며, 설정의 적용을 위해 프로세스를 다시 시작해야 할 수도 있다. 이러한 이유로 최후의 방법으로만 사용하는 것을 권장한다.

[ResourceQuota](https://kubernetes.io/docs/concepts/policy/resource-quotas/), [NetworkPolicy](https://kubernetes.io/docs/concepts/services-networking/network-policies/), [RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)와 같은 policy API는 선언적으로 구성된 정책 설정을 제공하는 k8s 내장 API다. 내장 정책 API는 po 등 다른 k8s 리소스와 동일한 규칙을 따른다. [stable](https://kubernetes.io/docs/reference/using-api/#api-versioning) 정책 API를 사용하면 다른 k8s API와 마찬가지로 [defined support policy](https://kubernetes.io/docs/reference/using-api/deprecation-policy/)을 받을 수 있다. 이러한 이유로 정책 API에 대한 사용이 설정 파일, CLI argument보다 권장된다.

## Extensions
extension은 k8s와 integrate되어 새로운 유형 또는 종류의 하드웨어를 지원하도록 k8s를 확장하는 소프트웨어 구성 요소다.

많은 cluster 관리자들은 호스팅되거나 배포된 k8s를 사용한다. 이러한 cluster에는 extension이 사전 설치되어 있다. 따라서 대부분의 k8s 사용자들은 extension을 따로 설치할 필요가 없으며 새로운 extension을 개발해야 하는 경우는 더 적다.

### Extension patterns
k8s는 client 프로그램을 작성함으로써 자동화될 수 있도록 설계됐다. k8s API를 read/write하는 모든 프로그램은 유용한 자동화를 제공할 수 있다. 자동화는 cluster 내부 또는 외부에서 실행될 수 있다. 이 문서의 지침을 따르면 고가용성이 높고 견고한 자동화를 작성할 수 있다. 일반적으로 자동화는 호스팅된 cluster 또는 관리형 설치를 포함한 모든 k8s cluster에서 작동한다.

k8s와 잘 작동하는 client 프로그램을 작성하기 위한 특정 패턴(controller pattern이라고 부름)이 있다. controller는 일반적으로 object의 `.spec`을 읽고, 필요에 따라 작업을 수행한 후 object의 `.status`를 업데이트한다.

controller는 k8s API의 client다. k8s가 client이고 원격 서비스를 호출할 때 k8s는 이를 웹훅(webhook)이라고 한다. 원격 서비스는 웹훅 백엔드(webhook backend)라고 한다. 사용자 정의 controlelr와 마찬가지로 webhook은 장애 지점을 추가한다.

> **Note**:  
> k8s 외부에서는 일반적으로 "webhook"은 asynchronous notification 메커니즘을 가리키며 웹훅 호출은 다른 시스템이나 구성 요소에 대한 단방향 notification 역할을 수행한다. k8s 생태계에서는 synchronous HTTP 호출도 종종 "webhook"으로 설명한다.

웹훅 모델에서는 k8s가 원격 서비스로 네트워크 요청을 보낸다. 이와 다르게 binary plugin 모델에서는 k8s가 binary 프로그램을 실행한다. binary plugin은 kubelet(예: [CSI storage plugin](https://kubernetes-csi.github.io/docs/), [CNI network plugin](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/))과 kubectl([Extend kubectl with plugins](https://kubernetes.io/docs/tasks/extend-kubectl/kubectl-plugins/))에서 사용된다.

### Extension points
![](https://kubernetes.io/docs/concepts/extend-kubernetes/extension-points.png)

#### Key to the figure 

1. 사용자는 주로 kubectl을 사용해 k8s API와 상호작용한다. 플러그인은 client의 동작을 customization한다. 다양한 client에 적용할 수 있는 일반적인 extension과 kubectl extension이 있다.

## Client extensions
kubectl의 플러그인은 특정 하위 명령어의 동작을 추가하거나 대체하는 별도의 binary 파일이다. 또한 kubectl 도구는 [credential plugins](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#client-go-credential-plugins)과 통합할 수 있다. 이러한 extension은 개별 사용자의 로컬 환경에만 영향을 미치므로 모든 사용자에 대한 정책을 강제할 수는 없다.

kubectl 도구의 플러그인은 [Extend kubectl with plugins](https://kubernetes.io/docs/tasks/extend-kubectl/kubectl-plugins/)를 참고한다.


## API extensions
### Custom resource definitions
### API aggregation layer
### Combining new APIs with automation
### Changing built-in resources
API access extensions
Authentication
Authorization
Dynamic admission control
Infrastructure extensions
Device plugins
Storage plugins
Network plugins
Kubelet image credential provider plugins
Scheduling extensions
