cr은 k8s API extension이다. 이 페이지에서는 k8s cluster에 cr을 추가해야 하는 시기와 standalone 서비스를 사용해야 하는 시기에 대해 설명한다. 또한 cr을 추가하는 두 가지 방법과 그 사이에서 선택하는 방법에 대해 설명한다.

## Custom resources
resource는 특정 종류의 API object 집합을 저장하는 k8s API의 엔드포인트다. 예를 들어, 내장된 po resource에는 po object의 집합이 포함되어 있다.

cr은 기본 k8s 설치 시 제공되지 않는 k8s API extension이다. 그러나 많은 핵심 k8s 기능이 cr을 사용해 k8가 더 모듈화되고 있다.

cluster 실행 중에 cr을 등록할 수 있으며 cluster와 독립적으로 cr을 업데이트 할 수 있다. cr이 설치되면 사용자는 kubectl을 통해 해당 object를 생성하고 접근할 수 있다.

## Custom controllers
cr을 사용해 구조화된 데이터를 저장하고 검색할 수 있다. 하지만 cr을 custom controller와 사용하면 cr은 진정한 의미의 declarative AIP를 제공한다.

k8s의 [declarative AIP](https://kubernetes.io/docs/concepts/overview/kubernetes-api/)는 책임의 분리를 강제한다. 사용자는 resource의 desired state를 선언한다. k8s controller는 현재 k8s object의 state을 desired state와 동기화한다. 이는 직접 서버에 지시하는 명령형 API와 대조된다.

cluster의 lifecycle과 독립적으로 실행 중인 cluster에 custom controller를 배포하고 업데이트할 수 있다. custom controller는 모든 종류의 resource와 동작할 수 있지만 특히 cr과 결합될 때 효과적이다. [Operator pattern](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/)은 cr과 custom controller를 결합한다. custom controller를 사용해 특정 애플리케이션에 대한 domain knowledge를 k8s API로 녹여낼 수 있다.

## Should I add a custom resource to my Kubernetes cluster?

### Declarative APIs

## Should I use a ConfigMap or a custom resource?

## Adding custom resources
k8s는 cr 추가를 위한 두 가지 방법을 제공한다.
- crd는 프로그래밍 없이 간단하게 생성할 수 있다.
- API Aggregation은 프로그래밍이 필요한 대신 데이터의 저장 방식, API 버전 간 변환과 같이 API의 동작을 더 많이 제어할 수 있다.

k8s는 다양한 사용자의 요구를 충족시키기 위해 이 두 가지 옵션을 제공한다. 이를 통해 사용의 편의성이나 유연성이 어느 쪽도 희생되지 않는다.

 aggregated API는 kube-apiserver 뒤에 있는 추가 API server로 kube-apiserver가 proxy 역할을 수행한다. 이러한 구조를 AA(API Aggregation)라고 한다. 사용자 입장에서는 단순히 k8s API가 확장된 것처럼 보일 뿐이다.

crd(Custom Resource Definitions)는 사용자가 새로운 유형의 resource를 추가 API server 없이 생성할 수 있다. CRD를 사용하기 위해 api aggregation을 이해할 필요가 없다.

위 두 가지 방식의 차이점과는 별개로 k8s가 내장 지원하는 resource와 구분하기 위해 cr이라고 부른다.

> **Note**:  
> cr을 애플리케이션, 사용자, 모니터링 데이터 저장소로 사용하면 안된다. k8s api 내에 애플리케이션 데이터를 저장하는 아키텍처 디자인은 너무 결합된 디자인을 나타낸다.
>
> 아키텍처적으로 [cloud native](https://www.cncf.io/about/faq/) 애플리케이션 아키텍처는 구성 요소 간의 느슨한 결합을 선호한다. workload의 일부가 루틴 작업에 대한 백엔드 서비스를 필요로 할 경우 해당 백엔드 서비스를 구성 요소로 실행하거나 외부 서비스로 사용해야 한다. 이렇게 함으로써 워크로드가 일반 작업에 대해 k9s API에 의존하지 않게 된다.

## CustomResourceDefinitions

## API server aggregation

Choosing a method for adding custom resources
Comparing ease of use
Advanced features and flexibility
Common Features
Preparing to install a custom resource
Third party code and new points of failure
Storage
Authentication, authorization, and auditing
Accessing a custom resource