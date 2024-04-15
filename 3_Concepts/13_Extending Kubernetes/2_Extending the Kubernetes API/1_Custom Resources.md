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
아래 내용에 해당할 경우 cf를 사용한다.
- mysql.conf, pom.xml과 같이 파일 포맷이 문서화가 이미 잘 된 경우
- cm에 전체 설정을 관리하길 원하는 경우
- 설정 파일의 목적이 po 내 실행되는 프로그램일 경우
- 파일을 사용하는 주체가 k8s API보다 po의 환경 변수 또는 파일을 선호하는 경우
- 파일이 변경됨에 따라 deploy가 rolling update를 수행하길 원하는 경우

> **Note**:  
> 민감 정보는 secret을 이용한다.

아래 내용에 해당할 경우 cr(crd 또는 AA)를 사용한다.
- 새로운 resource 생성, 업데이트를 위해 k8s client library, CLI를 사용하길 원하는 경우
- `kubectl get my-object object-name`과 같이 kubectl 명령어를 통해 지원하길 원하는 경우
- 새로운 object에 대한 업데이트를 추적하는 자동화를 구축하고 다른 object에 대한 CRUD를 수행하길 원하는 경우
- object에 대한 업데이트를 처리하는 자동화를 작성하길 원하는 경우
- `.spec`, `.status`, `.metadata`와 같이 k8s API convention을 사용하길 원하는 경우
- 제어하는 resource 집합에 대해 추상화를 원하는 경우

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
[CustomResourceDefinition](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/) API resource를 사용해 cr을 정의할 수 있다. crd object를 정의하면 지정한 이름과 스키마를 가진 새로운 cr이 생성된다. k8s API는 cr을 처리하며 cr을 위한 저장소를 제공한다. crd object의 이름은 유효한 [DNS subdomain name](https://kubernetes.io/docs/concepts/overview/working-with-objects/names#dns-subdomain-names)이어야 한다.

crd를 사용하면 cr을 처리하기 위해 별도의 API server를 개발할 필요는 없지만 [API server aggregation](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/#api-server-aggregation)보다 유연성 및 확장성이 떨어진다.

에제는 [https://github.com/kubernetes/sample-controller](https://github.com/kubernetes/sample-controller)을 참고한다.

새로운 사용자 지정 리소스를 등록하고 해당 새로운 리소스 유형의 인스턴스를 다루며 이벤트를 처리하기 위해 컨트롤러를 사용하는 예제는 사용자 지정 컨트롤러 예제를 참조하십시오.

## API server aggregation
일반적으로 k8s API의 각 resource는 REST 요청을 처리하고 object의 persistent storage을 관리하는 코드가 필요하다. kube-apiserver는 po, svc와 같은 내장된 resource를 처리하며 crd를 통해 생성된 cr도 처리할 수도 있다.

aggregation layer을 사용하면 cr을 처리하도록 구현된 자체 API server를 개발, 배포할 수 있다. kube-apiserver는 요청을 자체 API server로 위임하기 때문에 모든 client가 사용할 수 있다.

## Choosing a method for adding custom resources
crd는 사용하기 편한 반면, AA는 더 유연하다. 사용자 입장에서 필요에 따른 방법을 사용하면 된다.

일반적으로 crd는 아래 경우에 사용한다.
- 적은 수의 필드가 존재하는 경우
- 회사 내에서 사용하는 경우, 소규모 open-source project로 사용하는 경우

### Comparing ease of use

### Advanced features and flexibility

### Common Features

## Preparing to install a custom resource
cluster에 cr를 추가하기 전에 주의해야 할 사항이 있다.

### Third party code and new points of failure
crd를 생성하는 동안 새로운 point of failure(예를 들어 직접 kube-apiserver에서 third party code를 실행하는 것)가 추가되지 않는다. 패키지(예를 들어 chart)나 다른 installation bundle은 새로운 crd와 비즈니스 로직을 구현하는 third-party code에 대한 deploy를 포함한다.

AA server를 설치하는 것은 항상 새로운 deploy를 실행하는 것을 포함한다.

### Storage
cr은 cm과 동일한 방법으로 storage 공간을 소비한다. 너무 많은 cr를 생성하는 것은 API server의 storage 공간을 부족하게 할 수도 있다.

AA server는 kube-apiserver와 동일한 storage를 사용하기 때문에 동일한 문제가 발생할 수 있다.

### Authentication, authorization, and auditing
crd는 kube-apiserver의 내장된 resource와 동일한 authentication, authorization, audit log를 사용한다.

authorization을 위해 rbac를 사용하는 경우 대부분의 rbac role을 새로운 reousrce에 대한 접근을 부여하지 않는다(cluster-admin role, *를 사용하는 role 제외). 그렇기 때문에 새로운 resource에 대한 접근을 위해 명시적으로 부여해야 한다. crd, AA는 종종 새로운 role과 같이 제공되기도 한다.

AA server는 kube-apiserver와 동일한 authentication, authorization, audit log를 사용한다.

## Accessing a custom resource
k8s [client libraries]()를 사용해 cr에 접근할 수 있다. 모든 client library가 cr을 지원하는 것은 아니다. 

아래 방법을 사용해 cr에 접근할 수 있다.
- kubectl
- k8s dynamic client
- 직접 작성한 REST client
- [Kubernetes client generation tools](https://github.com/kubernetes/code-generator)를 사용해 생성된 client(generating one is an advanced undertaking, but some projects may provide a client along with the CRD or AA)