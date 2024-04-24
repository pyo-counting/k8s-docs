k8s는 확장성이 좋다. 그렇기 때문에 사용자가 k8s 프로젝트 코드를 직접 fork 하거나 patch 제안할 필요가 거의 없다.

이 가이드는 k8s cluster customization 옵션에 대해 설명한다. 사용자의 요구 사항에 맞도록 k8s cluster를 조정하는 방법을 이해하길 원하는 운영자에게 도움이 될 수 있다. 플랫폼 개발자, k8s 프로젝트 contributor는 extension point, pattern에 대해 확인할 수 있다.

customization은 크게 CLI argument, 로컬 설정 파일, API resource에 대한 변경을 포함하는 configuration과 추가 프로그램 실행, 네트워크 서비스 실행을 포함하는 extension이 있다. 여기서는 주로 extension에 대해 설명한다.

## Configuration
설정 파일, CLI argument는 각 구성요소의 documentation을 확인한다.
- kube-apiserver
- kube-controller-manager
- kube-scheduler
- kubelet
- kube-proxy

CLI argument, 설정 파일은 항상 변경할 수 있는 것은 아니다. 변경이 가능하더라도 일반적으로 이는 cluster 운영자만 변경할 수 있다. 또한 이러한 변경 사항은 향후 k8s 버전에서 변경될 수 있으며, 설정의 적용을 위해 프로세스를 다시 시작해야 할 수도 있다. 이러한 이유로 최후의 방법으로만 사용하는 것을 권장한다.

[ResourceQuota](https://kubernetes.io/docs/concepts/policy/resource-quotas/), [NetworkPolicy](https://kubernetes.io/docs/concepts/services-networking/network-policies/), [RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)은 선언적으로 구성된 정책 설정을 제공하는 k8s 내장 API다. 내장 정책 API는 po 등 다른 k8s 리소스와 동일한 규칙을 따른다. [stable](https://kubernetes.io/docs/reference/using-api/#api-versioning) 정책 API를 사용하면 다른 k8s API와 마찬가지로 [defined support policy](https://kubernetes.io/docs/reference/using-api/deprecation-policy/)을 받을 수 있다. 이러한 이유로 정책 API에 대한 사용이 설정 파일, CLI argument보다 권장된다.

## Extensions
extension은 k8s와 통합되어 새로운 유형 또는 종류의 하드웨어를 지원하도록 k8s를 확장하는 소프트웨어 구성 요소다.

많은 cluster 관리자들은 호스팅되거나 배포된 k8s를 사용한다. 이러한 cluster에는 보통 extension이 사전 설치되어 있다. 따라서 대부분의 k8s 사용자들은 extension을 따로 설치할 필요가 없으며 새로운 extension을 개발해야 하는 경우는 더 적다.

### Extension patterns
k8s는 client 프로그램을 작성함으로써 자동화될 수 있도록 설계됐다. k8s API를 read/write하는 모든 프로그램은 유용한 자동화를 제공할 수 있다. 자동화는 cluster 내부 또는 외부에서 실행될 수 있다. 이 문서의 지침을 따르면 고가용성이 높고 견고한 자동화를 작성할 수 있다. 일반적으로 자동화는 호스팅된 cluster 또는 관리형 설치를 포함한 모든 k8s cluster에서 작동한다.

k8s와 잘 작동하는 client 프로그램을 작성하기 위한 특정 패턴(controller pattern이라고 부름)이 있다. controller는 일반적으로 object의 `.spec`을 읽고, 필요에 따라 작업을 수행한 후 object의 `.status`를 업데이트한다.

controller는 k8s API의 client다. k8s가 client이고 원격 서비스를 호출할 때 k8s는 이를 웹훅(webhook)이라고 한다. 원격 서비스는 웹훅 백엔드(webhook backend)라고 한다. 사용자 정의 controller와 마찬가지로 webhook은 장애 지점을 추가한다.

> **Note**:  
> k8s 외부에서는 일반적으로 "webhook"은 asynchronous notification 메커니즘을 가리키며 웹훅 호출은 다른 시스템이나 구성 요소에 대한 단방향 notification 역할을 수행한다. k8s 생태계에서는 synchronous HTTP 호출도 종종 "webhook"으로 설명한다.

웹훅 모델에서는 k8s가 원격 서비스로 네트워크 요청을 보낸다. 이와 다르게 binary plugin 모델에서는 k8s가 binary 프로그램을 실행한다. binary plugin은 kubelet(예: [CSI storage plugin](https://kubernetes-csi.github.io/docs/), [CNI network plugin](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/))과 kubectl([Extend kubectl with plugins](https://kubernetes.io/docs/tasks/extend-kubectl/kubectl-plugins/))에서 사용된다.

### Extension points
아래 그림은 k8s cluster 구성요소 별 extension point를 보여준다.

![](https://kubernetes.io/docs/concepts/extend-kubernetes/extension-points.png)

#### Key to the figure 
1. 사용자는 주로 kubectl을 사용해 k8s API와 상호작용한다. 플러그인은 client의 동작을 customization한다. 다양한 client에 적용할 수 있는 일반적인 extension과 kubectl extension이 있다.
2. kube-apiserver는 모든 요청을 처리한다. kube-apiserver의 extension point는 요청을 인증하거나, 내용에 따라 차단하거나 내용을 편집, 삭제 처리하는 것을 허용한다.
3. kube-apiserver는 다양한 종류의 resource를 제공한다. po와 같이 내장된 resource는 k8s 프로젝트에 의해 정의되며 변경할 수 없다. 대신 API extension을 통해 cr을 추가할 수 있다.
4. k8s kube-scheduler는 po를 실행할 no를 결정한다. scheduling을 확장하기 위한 몇 가지 방법이 있다.
5. k8s의 대부분 동작은 kube-apiserver의 client인 controller라는 프로그램으로 구현된다. controller는 cr과 사용될 수도 있다.
6. kubelet은 no에 실행되며 cluster 네트워크 상에서 po가 자신의 ip를 갖는 가상 서버처럼 동작하도록 지원한다. network plugin은 po 네트워킹의 다른 부분을 허용한다.
7. device plugin을 사용하여 custom 하드웨어나 다른 특수 노드 로컬 구성요소를 통합하고 이를 cluster에서 실행 중인 po에 제공할 수 있다. kubelet에는 device plugin과 함께 작업할 수 있는 지원가 포함되어 있다.

또한 kubelet은 po과 그들의 container를 위한 volume을 mount하고 mount를 해제한다. storage plugin을 사용해 새로운 종류의 storage와 다른 volume 유형에 대한 지원을 추가할 수 있다.

#### Extension point choice flowchart
어디서부터 시작해야할지 모르겠다면 아래 flowchart가 도움이 될 수 있다.

![](https://kubernetes.io/docs/concepts/extend-kubernetes/flowchart.svg)

## Client extensions
kubectl의 플러그인은 특정 하위 명령어의 동작을 추가하거나 대체하는 별도의 binary 파일이다. 또한 kubectl 도구는 [credential plugins](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#client-go-credential-plugins)과 통합할 수 있다. 이러한 extension은 개별 사용자의 로컬 환경에만 영향을 미치므로 모든 사용자에 대한 정책을 강제할 수는 없다.

kubectl의 플러그인은 [Extend kubectl with plugins](https://kubernetes.io/docs/tasks/extend-kubectl/kubectl-plugins/)를 참고한다.

## API extensions
### Custom resource definitions
k8s에 새로운 controller, 애플리케이션 설정 object, 기타 declarative API를 정의하고 kubectl과 같은 k8s 도구를 사용해 관리하길 원하는 경우 k8s에 cr을 추가하는 것을 고려한다.

cr에 대한 자세한 내용은 [Custom Resources](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)를 참고한다.

### API aggregation layer
k8s의 [API Aggregation Layer](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/apiserver-aggregation/)를 사용해 [metrics](https://kubernetes.io/docs/tasks/debug/debug-cluster/resource-metrics-pipeline/)와 같이 추가 서비르르 k8s API와 통합할 수 있다.

### Combining new APIs with automation
cr API와 control loop의 조합을 controller pattern이라고 한다. 그리고 controller가 desired state에 기반해 운영자 대신 인프라를 배포하는 경우 해당 controller는 operator pattern를 따른다고 한다. operator pattern은 특정 애플리케이션을 관리하기 위해 사용한다. 일반적으로 이러한 애플리케이션은 state를 유지하며 관리 방법에 주의를 요하는 애플리케이션을 의미한다.

그리고 storage와 같이 다른 resource를 관리하거나 접근 제어 제한과 같은 정책을 정의하는 custom API와 control loop를 만들 수 있다.

### Changing built-in resources
cr를 추가해 k8s API를 확장할 때 추가된 resource는 항상 새로운 API 그룹에 속하게된다. 기존 API 그룹을 대체하거나 변경할 수 없다. 새로운 API를 추가하는 것은 기존 API 동작에 직접적으로 영향을 주지 않는다. 반면에 API Access Extension은 예외다.

## API access extensions
kube-apiserver에 요청이 도달하면 authentication -> authorization -> 여러 유형의 admission control이 적용된다(일부 요청은 authentication을 거치지 않는 경우도 있다). 자세한 내용은 [Controlling Access to the Kubernetes API](https://kubernetes.io/docs/concepts/security/controlling-access/)을 참고한다.

k8s의 authentication, authorization 각 단계는 extension point를 제공한다.

### Authentication
[Authentication](https://kubernetes.io/docs/reference/access-authn-authz/authentication/)은 모든 클라이언트 요청의 header나 certificate를 사용자 이름으로 매핑한다.

쿠버네티스는 기본적으로 지원하는 여러 인증 방법을 가지고 있습니다. 인증 프록시 뒤에 있을 수도 있고, 토큰을 인증 웹 훅(Authorization: Header)에서 사용자의 요구에 맞지 않을 경우 원격 서비스(인증 웹 훅)로 보낼 수도 있습니다.

k8s는 여러 내장 authentication을 제공한다. 필요에 따라 authentication proxy 뒤에 두거나, 요구에 맞지 않을 경우 token(`Authorization:` header)을 원격 서비스로 전송([authentication webhook](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#webhook-token-authentication))할 수도 있다.

### Authorization
[Authorization](https://kubernetes.io/docs/reference/access-authn-authz/authorization/) 특정 사용자가 API resource를 read, write 등의 작업을 수행할 수 있는지 여부를 결정한다. 이는 전체 resource level에서 동작하며 임의의 object 필드를 기반으로 구분하지 않는다.

내장된 authorization이 필요하는 요구 사항을 충족하지 못하는 경우 [authorization webhook](https://kubernetes.io/docs/reference/access-authn-authz/webhook/)을 사용해 custom code를 호출해 authorization 결정을 내릴 수 있다.

### Dynamic admission control
요청이 인가된 후, write 작업은 [Admission Control](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/)단계를 거친다. 내장 단계 외에도 여러 extension이 있다.

- [Image Policy webhook](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#imagepolicywebhook)은 container에서 사용할 수 있는 image를 제한한다.
- To make arbitrary admission control decisions, a general Admission webhook can be used. Admission webhooks can reject creations or updates. Some admission webhooks modify the incoming request data before it is handled further by Kubernetes.

## Infrastructure extensions
### Device plugins
[device plugin](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/)은 no가 새로운 resource 유형(내장 memory, cpu 외)을 찾는 것을 허용한다.

### Storage plugins
container stroage interface(CSI) 플러그인은 k8s를 새로운 종류의 volume에 대한 지원을 확장하는 방법을 제공한다. 이러한 volume은 내구성 있는 외부 storage로 백업되거나 ephemeral storage를 제공하거나, 파일 시스템 패러다임을 사용하여 정보에 대한 읽기 전용 인터페이스를 제공할 수 있다.

또한 k8s에는 CSI를 대신하여 k8s v1.23부터 사용이 중단된 [FlexVolume](https://kubernetes.io/docs/concepts/storage/volumes/#flexvolume) 플러그인을 지원한다.

FlexVolume 플러그인을 사용하면 k8s에서 기본적으로 지원하지 않는 볼륨 유형을 mount할 수 있다. FlexVolume storage를 사용하는 po를 실행하면 kubelet이 volume을 mount하기 위해 binary 플러그인을 호출한다. [FlexVolume](https://github.com/kubernetes/design-proposals-archive/blob/main/storage/flexvolume-deployment.md) design proposal에는 이 접근 방식에 대한 자세한 내용이 있다.

storage plugin를 위한 [Kubernetes Volume Plugin FAQ for Storage Vendors](https://github.com/kubernetes/community/blob/master/sig-storage/volume-plugin-faq.md#kubernetes-volume-plugin-faq-for-storage-vendors)에는 storage plugin에 관한 일반 정보가 포함되어 있다.

### Network plugins
k8s cluster no는 po 네트워크, k8s 네트워크 모델의 다른 측면을 지원하기 위해 network plugin이 필요하다.

[Network plugins](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/)는 k8s의 다양한 네트워킹 토폴로지와 기술을 사용할 수 있도록 한다.
### Kubelet image credential provider plugins
kubelet image credential provider는 kubelet의 플러그인으로 image registry credential을 동적으로 검색하는 역할을한다. credential은 설정과 일치하는 container image registry에서 image를 가져올 때 사용된다,

이 플러그인은 외부 서비스와 통신하거나 로컬 파일을 사용해 credential을 얻을 수 있다. 이를 통해 kubelet은 각 registry에 대한 static credential이 필요하지 않으며 다양한 인증 방법과 프로토콜을 지원할 수 있다.

플러그인 설정 세부 정보는 [Configure a kubelet image credential provider](https://kubernetes.io/docs/tasks/administer-cluster/kubelet-credential-provider/)을 참고한다..

## Scheduling extensions
kube-scheduler는 po를 감시하고 po을 no에 할당하는 특별한 유형의 controller다. 기본 kube-scheduler는 완전히 대체할 수 있으며 다른 k8s 구성 요소를 계속 사용하거나 [multiple schedulers](https://kubernetes.io/docs/tasks/extend-kubernetes/configure-multiple-schedulers/)를 동시에 실행할 수 있다.

이는 매우 중요한 작업이며 거의 모든 k8s 사용자가 kube-scheduler를 수정할 필요가 없다.

[scheduling plugins](https://kubernetes.io/docs/reference/scheduling/config/#scheduling-plugins)을 활성화하거나 서로 다른 이름의 [scheduler profiles](https://kubernetes.io/docs/reference/scheduling/config/#multiple-profiles)를 연결할 수도 있다. 그리고 하나 이상의 kube-scheduler의 [extension point](https://kubernetes.io/docs/concepts/scheduling-eviction/scheduling-framework/#extension-points)와 통합하는 자체 플러그인을 작성할 수도 있다.

마지막으로 내장된 kube-scheduler 구성 요소는 원격 HTTP backend(scheduler extension)가 po에 대해 kube-scheduler가 선택한 no를 필터링하고/또는 우선 순위를 지정할 수 있도록 허용하는 [webhook](https://git.k8s.io/design-proposals-archive/scheduling/scheduler_extender.md)을 지원한다.

**Note**:  
> scheduler extender webhook을 통해 no 필터링, no 우선 순위를 변경할 수 있다. other extension points are not available through the webhook integration.