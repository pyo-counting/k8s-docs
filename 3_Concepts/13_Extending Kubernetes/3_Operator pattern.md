operator는 [custom resources](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)를 사용해 애플리케이션과 관련 구성요소를 관리하는 k8s 소프트웨어 extension이다. operator는 k8s의 원칙, 특히 [control loop](https://kubernetes.io/docs/concepts/architecture/controller/)를 따른다.

## Motivation
operator pattern은 애플리케이션 또는 애플리케이션과 관련된 구성요소 집합을 관리하는 운영자의 주요 업무를 포착하는 것을 목표로한다. 특정 애플리케이션을 관리하고 운영하는 사람은 해당 시스템의 동작 방식, 배포 방법, 문제 발생 시 해결 방법을 잘 안다.

k8s에서 workload를 실행하는 사람들은 반복적인 작업을 처리하기 위해 자동화를 하는 것을 좋아한다. operator pattern은 k8s가 제공하는 것 이상의 작업을 자동화하기 위해 코드를 작성하는 방법을 포착한다.

## Operators in Kubernetes
k8s는 기본적으로 자동화를 위해 디자인되었다. k8s 코어에는 내장된 자동화 기능을 지원한다. k8s를 사용해 워크로드 배포 및 실행을 자동화할 수 있으며, k8s가 해당 작업을 수행하는 방식도 자동화할 수 있다.

k8s의 operator pattern 개념을 통해 k8s 코드 자체를 수정하지 않고도 controller를 하나 이상의 custom resource에 연결해 cluster의 동작을 확장할 수 있다. operator는 custom resource의 controller 역할을 하는 k8s API 클라이언트다.

## An example operator
operator를 사용하해 자동화할 수 있는 몇 가지 예시는 다음과 같다.

- on-demand 애플리케이션 배포
- 애플리케이션의 상태를 백업하고 복원
-  데이터베이스 스키마 또는 추가 설정과 같은 관련 변경 사항에 따른 애플리케이션 코드 업그레이드 처리
- k8s API를 지원하지 않는 애플리케이션에 서비스를 publish함으로써 검색을 지원
- 클러스터의 전체 또는 일부에서 장애를 시뮬레이션하여 가용성 테스트
- 내부 멤버 선출 절차없이 분산 애플리케이션의 리더를 선택

operator 패턴에 대한 자세한 예시는 다음과 같다.
1. 클러스터에 구성할 수 있는 SampleDB라는 cr
2. operator controller의 구성요소가 포함된 po의 실행을 보장하는 deploy
3. operator code의 container image
4. control plance을 쿼리해 어떤 SampleDB resource가 배포됐는지 추적하는 controller 코드
5. operator의 핵심은 API server에 배포된 resource와 실제 current state를 매칭시키는 방법을 알려주는 코드이다.
    - 새 SampleDB object를 추가하면 operator는 pvc을 설정하여 내구성있는 데이터베이스 스토리지, SampleDB를 실행하는 sts, 초기 구성을 처리하는 job을 제공한다.
    - SampleDB를 삭제하면 operator는 스냅샷을 생성한 다음 sts와 volume도 제거되었는지 확인한다.
6. operator는 주기적인 데이터베이스 백업도 관리한다. operator는 각 SampleDB resource 대해 데이터베이스에 연결하고 백업을 수행할 수 있는 po를 생성하는 시기를 결정한다. 이 po는 데이터베이스 연결 세부 정보 및 자격 증명이 있는 cm, secret에 의존한다.
7. operator는 관리하는 resource에 견고한 자동화를 제공하는 것을 목표로 하기 때문에 추가 지원 코드가 있다. 이 예제에서 코드는 데이터베이스가 이전 버전을 실행 중인지 확인하고, 업그레이드된 경우 이를 업그레이드하는 job 오브젝트를 생성한다.

## Deploying operators
operator를 배포하는 가장 일반적인 방법은 crd 정의, 관련 controller를 cluster에 추가하는 것이다. 컨테이너화된 애플리케이션을 실행하는 것처럼 controller는 일반적으로 control plane 외부에서 실행된다. 예를 들어 cluster에서 controller를 deploy로 실행할 수 있다.

## Using an operator
operator가 배포되면 operator가 사용하는 resource의 종류를 추가/수정/삭제해 사용한다. 위의 예에 따라 operator 자체에 대한 deploy를 설정한 후 아래 명령어를 수행한다.

```
kubectl get SampleDB                   # 구성된 데이터베이스 찾기
kubectl edit SampleDB/example-database # 일부 설정을 수동으로 변경하기
```

operator는 변경 사항을 적용하고 기존 서비스를 양호한 상태로 유지한다.

## Writing your own operator
ecosystem에 원하는 동작을 구현하는 operator가 없다면 직접 코딩할 수 있다.

또한 k8s API의 클라이언트 역할을 할 수 있는 모든 언어 / 런타임을 사용하여 operator(즉, controller)를 구현한다.

다음은 cloud native operator를 작성하는 데 사용할 수 있는 몇 가지 라이브러리와 도구들이다.

**Note**: 이 섹션은 k8s에 필요한 기능을 제공하는 third party 프로젝트와 관련이 있다. k8s 프로젝트 작성자는 third party 프로젝트에 책임이 없다.

- Charmed Operator Framework
- Java Operator SDK
- Kopf (Kubernetes Operator Pythonic Framework)
- kube-rs (Rust)
- kubebuilder 사용하기
- KubeOps (.NET 오퍼레이터 SDK)
- KUDO (Kubernetes Universal Declarative Operator)
- 웹훅(WebHook)과 함께 Metacontroller를 사용하여 직접 구현하기
- 오퍼레이터 프레임워크
- shell-operator