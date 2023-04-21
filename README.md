# Kubernetes
Kubernetes 학습

## 요약
- k8s의 worker node 구성 요소중 container 실행을 담당하는 kublet은 container로 실행할 수 없다.
- linux container는 격리를 위해 namespace, cgroup(control group) 기술을 사용한다.
  - namespace:
    - mnt: mount points
    - user: user and group ids
    - uts(unix time sharing): hostname, nis domain name
    - pid: process ids
    - network: network devices, stacks, ports, etc
    - ipc(inter process communication): system V IPC, POSIX message queues
    - control groups: control group root directory
  - cgroup
    - 메모리, cpu, 네트워크 대역폭 등
- 이론적으로 container는 모든 리눅스 시스템에서 실행될 수 있지만 호스트의 커널을 사용하기 때문에 커널 버전에 영향을 받는다. 뿐만 아니라 특정 하드웨어 아키텍쳐용(x86, ARM 등)으로 만들어진 image는 해당 아키텍쳐에서만 실행될 수 있다.
- kubernetes component
  - kubectl은 k8s control plane의 구성 요소 API server와 요청/응답을 통해 통신한다.
  - addon은 k8s resource(ds, deploy 등)를 이용하여 클러스터 기능을 구현한다. 이들은 클러스터 단위의 기능을 제공하기 때문에 addon에 대한 resource는 kube-system ns에 속한다.
  - kube-proxy는 클러스터의 각 no에서 실행되는 네트워크 프록시로 k8s의 svc 개념의 구현부이다.
- docker 기준 k8s는 po내 모든 container가 동일한 linux namespace를 공유하도록 docker를 설정한다. 파일 시스템의 경우 image에 저장되어 있어 기본적으로 완전히 분리된다.
  - 동일한 namespace: network, uts, ipc
  - 다른 namespace: uid, pid, mnt
- po 매니페스트 중 spec.containers[\*].ports[\*].containerport는 container가 노출하는 포트 정보를 명시하기만 할 뿐 다른 기능은 없다. 이를 생략한다고 해서 포트를 통해 po에 연결할 수 있는 여부에 영향을 미치지 않는다.
- po는 k8s에서 배포할 수 있는 가장 작은 단위로, 이미 실행 중인 po 내에 container를 배포할 수 없다. 물론 po에 포함된 container가 po 내에서 재시작 될 수 있다.
- static po는 다른 po들(control plane이 담당)과 다르게 해당 no의 kubelet이 직접 관리한다. 그리고 static po에 대한 object를 API server에 게시한다(API server를 통해 object를 변경하는 것은 불가능). 즉 static po가 특정 kubelet에 종속되어 있지만 API server를 통해 조회가 가능하다. static po의 .spec에서는 다른 API object를 참조할 수 없다(예를 들어 sa, cm, secret 등).
- po는 control plane에서 관리하며, scheduler룰 통해 어떤 no에서 po가 실행될지 결정한다. no의 kubelet은 해당 po의 container를 실행 및 관리할 책임이 있다.
- container 로그는 하루 단위, 10MB 크기 기준으로 롤링(rolling)된다.
- annotation은 key-value으로 label과 유사하지만 식별 정보로 사용되지 않는다. 즉 label과 같이 selector 기능은 없다. annotation은 주로 해당 리소스에 대한 정보를 나타내는데 사용된다.
- k8s namespace는 리소스 이름의 범위를 제공한다. 뿐만 아니라 리소스를 격리하는 것외에도 특정 사용자가 지정된 리소스에 접근할 수 있도록 허용하고, 개별 사용자가 사용할 수 있는 컴퓨팅 리소스를 제한하는 데에도 사용된다. 하지만 리소간 격리는 제공하지 않는다. 즉 서로 다른 namepsace에 존재하는 리소스더라도 통신할 수 있다. 네트워크 격리는 k8s와 함께 배포되는 네트워킹 솔루션에 따라 다르다.
- 대부분의 리소스 이름은 RFC 1035 (domain name)에 지정된 규칙을 준수해야 한다. 즉, 글자, 숫자, 대시, 점을 포함할 수 있다. 하지만 몇몇 리소스는 점을 포함할 수 없다.
- k8s에서 추가하는 annotation, label을 식별하기 위해 kubernetes.io/, k8s.io/ label을 예약했다. k8가 자동으로 붙이는 label
  - ns
    - kubernetes.io/metadata.name: NamespaceDefaultLabelName feature gate가 활성화됐을 경우 추가되며 ns의 이름을 갖는다.
- owner reference는 k8s resource 간의 종속 관계를 나타낸다. k8s는 object 삭제 시 label이 아닌 owner reference를 사용해 종속 관계에 대한 cascading deletion(background 또는 foreground)을 수행한다. 
- container의 생명주기와 관련해 hook을 제공하며, handler를 구현함으로써 hook에 대한 이벤트를 처리할 수 있다.
  - PostStart: container와 비동기적으로 실행된다. 하지만 PreStart가 완료되지 않으면 container는 running state에 도달할 수 없다.
  - PreStop: container에 TERM 시그널이 전송되기 전에 실행된다. po의 terminationGracePeriodSeconds 설정보다 PreStop, TERM signal 전송 및 처리 시간이 더 오래걸이면 container는 비정상 종료 될 수도 있다.
- k8s no resource는 해당 호스트의 image 목록도 저장한다. 이는 k describe no 명령어를 사용해 조회되지 않는 정보다.
- .spec.selctor는 .spec.type이 ClusterIP, NodePort, LoadBalancer일 떄만 적용됨. ExternalName일 때는 무시된다고 documentation에 명시되어 있지만, ep는 생성됨을 확인(물론 프록시는 되지 않음). 
- .spec.selector가 없는 svc는 일반적인 svc와 동일하게 동작한다. 하지만 ep를 자동으로 생성하지 않으며 사용자가 직접 생성/관리해야 하는 책임이 있다.
- .spec.type을 ExternalName으로 바꾸거나 반대로 바꾸지 않는 이상 svc가 수정되더라도 clusterIP는 고정 값이다.
- .spec.clusterIP는 .spec.type이 NodePort, ClusterIP, LoadBalancer일 때 자동 할당되거나 사용자가 설정할 수 있다.
- .spec.clusterIP가 None: headless svc를 의미(cluster ip 미할당). kube-proxy가 해당 svc를 다루지 않기 때문에 프록시, 로드밸런싱되지 않는다. 즉 .spec.ports[*]를 명시해도 의미가 없다.
  - .spec.selector가 있으면 svc DNS lookup 시 pod들의 ip 목록(A 레코드)가 조회됨. ep도 생성됨. .spec.clusterIP: "None"이 아니면 오류. 조회된 pod의 ip와 po의 노출 po로 접근할 수 있다.
  - .spec.selector가 없으면 svc DNS lookup 시 아무것도 조회 안됨
- .spec.type이 ClusterIP: \<cluster ip>:\<port>로 접근 가능
- .spec.type이 NodePort: \<node ip>: \<node port>, \<cluter ip>:\<port>로 접근 가능 
- .spec.type이 ExternalName: .spec.externalName 필드 필수로 설정 필요. .spec.clusterIP 핃드를 명시하지 않거나 값을 ""으로 설정 필요. cluster ip, port 할당되지 않음. svc DNS lookup 시 CNAME 레코드 조회됨.

## 명령어
- kubectl get [RESOURCE] --field-selector

## 체크리스트
- po내 ports[*].hostPort에 사용된 port는 호스트 netstat 조회 시 보이지 않음. 하지만 type=ClusterIP svc로 expose 시 netstat에 조회됨
- svc externalIPs 설정 시, no의 IP로 svc 접근 가능
- local pv의 경우, pv 생성 시 디렉토리가 호스트 내 존재해야 함. 자동 생성되지 않음 확인
- hpa의 cpu 사용률 계산 시 po를 선택하는 방법 알아보기