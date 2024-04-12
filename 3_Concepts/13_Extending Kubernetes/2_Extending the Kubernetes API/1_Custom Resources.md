cr은 k8s API extension이다. 이 페이지에서는 k8s cluster에 cr을 추가해야 하는 시기와 standalone 서비스를 사용해야 하는 시기에 대해 설명한다. 또한 cr을 추가하는 두 가지 방법과 그 사이에서 선택하는 방법에 대해 설명한다.

## Custom resources
resource는 특정 종류의 API object 집합을 저장하는 k8s API의 엔드포인트다. 예를 들어, 내장된 po resource에는 po object의 집합이 포함되어 있다.

cr은 기본 k8s 설치 시 제공되지 않는 k8s API extension이다. 그러나 많은 핵심 k8s 기능이 cr을 사용해 k8가 더 모듈화되고 있다.

cluster 실행 중에 cr을 등록할 수 있으며 cluster와 독립적으로 cr을 업데이트 할 수 있다. cr이 설치되면 사용자는 kubectl을 통해 해당 object를 생성하고 접근할 수 있다.

## Custom controllers
cr을 사용해 구조화된 데이터를 저장하고 검색할 수 있다. 하지만 cr을 custom controller와 사용하면 cr은 진정한 의미의 declarative AIP를 제공한다.

k8s의 [declarative AIP](https://kubernetes.io/docs/concepts/overview/kubernetes-api/)는 책임의 분리를 강제한다. 사용자는 resource의 desired state를 선언한다. k8s controller는 현재 k8s object의 state을 desired state와 동기화한다. 이는 직접 서버에 지시하는 명령형 API와 대조된다.

cluster의 라이프사이클과는 독립적으로 실행 중인 cluster에 custom controller를 배포하고 업데이트할 수 있다. custom controller는 모든 종류의 resource와 동작할 수 있지만 특히 cr과 결합될 때 효과적이다. [Operator pattern](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/)는 cr과 custom controller를 결합한다. custom controller를 사용해 특정 애플리케이션에 대한 domain knowledge를 k8s API로 녹여낼 수 있다.