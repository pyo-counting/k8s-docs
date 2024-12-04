AWS와 같은 cloud를 사용하면 public, private, hybrid cloud에서 k8s를 실행할 수 있다. k8s는 구성 요소 간의 강한 coupling 없는 API-driven infrastructure를 지원한다.

cloud-controller-manager는 각 클라우드와 호환되는 제어 로직을 포함하는 k8s control plane 구성 요소다. cloud-controller-manager를 사용해 k8s cluster와 cloud provider의 API를 연결할 수 있으며 다른 구성 요소와 분리해서 관리할 수 있다.

k8s와 cloud 인프라 간의 상호 운용성 로직을 분리함으로써 cloud-controller-manager 구성 요소는 cloud provider가 k8s 프로젝트와 별개로 기능을 릴리즈할 수 있다.

cloud-controller-manager는 plugin mechanism을 사용해 서로 다른 cloud provider의 플랫폼을 k8s와 통합할 수 있도록 설계된다.

## Design
![](https://kubernetes.io/images/docs/components-of-kubernetes.svg)

cloud-controller-manager는 control plane에서 replica(보통 po)로 실행된다. 각 cloud-controller-manager는 단일 프로세스 내 여러 controller를 구현한다.

> **Note**:  
> cloud-controller-manager를 control plane의 구성 요소가 아닌 addon으로 실행할 수 있다.

## Cloud controller manager functions
cloud-controller-manager에 포함된 controller는 다음과 같다.

### Node controller
node controller는 클라우드 인프라에서 새로운 서버가 생성될 때 no 객체를 업데이트하는 역할을 수행한다. node controller는 cloud provider의 API를 사용해 실행 중인 호스트에 대한 정보를 가져온다. node controller는 다음과 같은 기능을 수행한다.
1. cloud provider API에서 가져온 서버의 고유 식별자로 no 객체를 업데이트한다.
2. no가 배포된 region, available resource와 같은 cloud 별 정보를 no object의 label, annotation으로 추가한다.
3. no의 hostname, network 주소를 가져온다.
4. no의 상태를 확인한다. no가 응답하지 않는 경우 controller는 cloud provider API를 사용해 서버의 상태(비활성/삭제/종료)를 확인한다. no가 cloud에서 삭제된 경우 controller는 해당 no 객체를 k8s cluster에서 삭제한다.

일부 cloud provider는 node controller, node lifecycle controller로 분리해서 구현하는 경우도 있다.

### Route controller
route controller는 k8s cluster의 다른 no에 있는 container가 서로 통신할 수 있도록 cloud 내에서 route를 구성한다.

cloud provider에 따라 route controller는 po 네트워크를 위해 ip 주소 CIDR block을 할당한다.

### Service controller
svc는 cloud에서 제공하는 load balancer, ip 주소, network packet filtering, target health check와 같은 cloud 구성 요소와 통합된다. service controller는 해당 구성 요소가 필요한 svc object를 생성할 때 cloud provider API와 상호 작용해 load balancer와 같은 인프라 구성 요소를 구성한다.

## Authorization
cloud-controller-manager가 위와 같은 다양한 동작을 수행하기 위해 k8s resource에 대한 권한 필요하다.

### Node controller
node controller는 no object에 대해서만 작업을 수행하며 모든 접근이 필요하다.

`v1/Node`
- get
- list
- create
- update
- patch
- watch
- delete

### Route controller
route controller는 no object의 생성을 읽고 적절한 route를 구성한다. 이를 위해 no object에 대한 get 접근이 필요하다.

`v1/Node`
- get

### Service controller
service controller는 svc object의 생성, 업데이트, 삭제를 감시하고 해당 svc에 대한 endpoint를 적잘하게 구성한다(endpointslice의 경우 kube-controller-manager가 필요에 따라 관리한다).

svc에 접근하기 위해 list, watch 접근이 칠요하다. 그리고 svc를 업데이트하기 위해 patch, update 접근이 칠요하다.

svc에 대한 endpoint object를 설정하기 위해 create, list, get, watch, update 접근이 필요하다.

`v1/Service`
- list
- get
- watch
- patch
- update

### Others
cloud-controller-manager 구현을 위해 event object 생성 접근 권한, 안전한 작동을 보장하기 위해 sa 생성 접근 권한이 필요하다.

`v1/Event`
- create
- patch
- update

`v1/ServiceAccount`
- create

cloud-controller-manager를 위한 RBAC clusterrole은 다음과 같다.
``` yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cloud-controller-manager
rules:
- apiGroups:
  - ""
  resources:
  - events
  verbs:
  - create
  - patch
  - update
- apiGroups:
  - ""
  resources:
  - nodes
  verbs:
  - '*'
- apiGroups:
  - ""
  resources:
  - nodes/status
  verbs:
  - patch
- apiGroups:
  - ""
  resources:
  - services
  verbs:
  - list
  - patch
  - update
  - watch
- apiGroups:
  - ""
  resources:
  - serviceaccounts
  verbs:
  - create
- apiGroups:
  - ""
  resources:
  - persistentvolumes
  verbs:
  - get
  - list
  - update
  - watch
- apiGroups:
  - ""
  resources:
  - endpoints
  verbs:
  - create
  - get
  - list
  - watch
  - update
```