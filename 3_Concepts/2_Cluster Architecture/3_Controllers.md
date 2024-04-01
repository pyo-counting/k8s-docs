로보틱스와 자동화에서 control loop는 시스템의 상태를 조절하는 종료되지 않는 loop이다.

예시: 실내 온도 조절기

사용자는 온도를 설정함으로써 사용자가 원하는 desired state를 온도 조절기에 알려준다. 실제 방의 온드는 current state다. 온도 조절기는 장비를 켜거나 끔으로써 current state를 desired state에 가깝게 유지하려고 한다.

k8s에서 controller는 클러스터의 상태를 관찰 한 다음 경우에 따라 생성 또는 변경을 요청하는 control loop다. 각 controller는 현재 클러스터 상태를 desired state와 최대한 동일하게 만들기 위해 노력한다.

## Controller pattern
controller는 적어도 1개의 k8s의 resource 타입을 추적한다. 이러한 object는 .spec 필드를 통해 desired state를 표현한다. 해당 resoure에 대한 controller는 object의 current state를 .sepc에 정의된 desired state에 가깝게 유지하기 위한 책임을 갖는다.

controller는 이를 위한 작업을 스스로 수행할 수 있다. 일반적으로 k8s에서는 controller가 API서버로 메시지를 전송한다. 관련된 내용은 아래에서 살펴본다.

### Control via API server
job controller는 k8s에 내장된 controller다. 내장된 controller는 API server와 상호작용함으로써 상태를 관리한다.

job은 po를 실행함으로써 작업을 수행하는 k8s resource다. 

job controller는 새로운 작업을 확인하면 클러스터 내 no들의 kubelet에서 해당 작업을 수행하기 위한 po를 실행하게한다. job controller는 직접 po 또는 container를 실행하지는 않는다. 대신 job controller는 API server에 po에 대한 생성 또는 삭제를 요청한다. 그러면 control plane의 다른 구성요소가 해당 요청에 대응해 작업을 완료시킨다.

job resource objec의 desired state는 .spec 필드에 명시된 작업을 수행하는 것이다. job controller는 해당 작업을 수행함으로써 job resource의 current state를 desired state에 가까워지도록 유지하기위해 노력한다.

뿐만 아니라 controller는 object의 설정을 업데이트 한다. 예를 들어 작업이 완료되면, job이 완료되었음을 나타내기 위해 Finished로 표시하도록 업데이트한다.

### Direct control
일부 controller는 클러스터 외부를 변경해야하는 경우도 있다.

예를 들어 클러스터 내에 충분한 no가 있을 수 있도록 유지하기 위한 controller를 사용하는 경우, 해당 controller는 필요할 떄 새로운 node를 설정할 수 있도록 클러스터 외부의 무언가를 필요로한다.

여기서 중요한 점은 controller가  desired state를 만드릭 위해 약간의 변화를 만들고, current state를 클러스터의 API server에 다시 보고한다는 것이다. 다른 control loop는 보고된 데이터를 관찰하고 이에 따른 작업을 수행할 수도 있다.

온도 조절기 예시와 같이 방이 매우 추우면 다른 controller가 서리 방지 히터를 켤 수도 있다. k8s 클러스터에서는 k8s 확장을 통해  IP address management tools, storage services, cloud provider API, 기타 서비스 등과 간접적으로 연동하여 이를 구현한다.


## Desired versus current state
k8s는 cloud-native 관점에서 시스템을 관찰하며 지속적인 변화에 대응할 수 있다.

클러스터는 언제든지 작업이 발생하고 자동으로 실패를 고칠수 있다. 이는 잠재적으로 클러스터가 안정적인 상태에 도달하지 못하는 것을 의미한다.

클러스터의 controller 실행 중이고 유용한 변경을 수행할 수 있는 한 전체 상태가 안정적인지 아닌지는 중요하지 않다.

## Design
디자인 원리에 따라 k8s는 클러스터 상태의 각 특정 측면을 관리하는 많은 controller를 사용한다. 가장 일반적으로 특정 control loop(controller)는 한 종류의 k8s resource의 desired state를 추적하며, desired state로 만들기 위해 다른 종류의 resource를 관리한다. 예를 들어, job controller는 job object와 새로운 job의 desired state를 만족하기 위해 po object를 추적한다. 이 경우 po는 job controller 생성하는 반면, job은 다른 controller가 생성한다.

control loop들로 연결 구성된 하나의 모놀리식(monolithic) 집합보다, 간단한 controller를 여러 개 사용하는 것이 유용하다. controller는 실패할 수 있으며 k8s는 이를 허용하도록 디자인되었다.

**Note**: 동일한 종류의 resource object를 만들거나 업데이트하는 여러 controller가 있을 수 있다. k8s controller는 관리하고 있는 resource에 연결된 resource에만 주의를 기울인다.

예를 들어, deploy와 job object를 가지고 있다고 가정하자. 두 resource 모두 po를 생성한다. job controller는 deploy가 생성한 po를 삭제하지 않는다. 이는 controller가 해당 po를 구별하기 위해 사용할 수 있는 정보(label)가 있기 때문이다.

## Ways of running controllers
k8s에는 kube-controller-manager 내부에서 실행되는 내장된 controller 집합이 있다. 이 내장 controller는 중요한 핵심 동작을 제공한다.

deploy controller와 job controller는 k8s의 자체("내장" 컨트롤러)로 제공되는 controller 예시이다. k8s를 사용하면 복원력(resilient)이 뛰어난 control plane를 구성하기 때문에 일부 내장 controller가 실패하더라도 다른 control plance의 일부가 작업을 이어서 수행한다.

control plance의 외부에서 실행하는 controller를 통해 k8s를 확장할 수 있다. 또는, 원하는 경우 새 controller를 직접 작성할 수 있다. 소유하고 있는 controller를 po 집합으로서 실행하거나 또는 k8s 외부에서 실행할 수 있다. 이는 해당 컨트롤러의 기능에 따라 달라질 수 있다.