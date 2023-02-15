# Kubernetes
Kubernetes 학습

## 요약
- k8s의 worker node 구성 요소중 container 실행을 담당하는 kublet은 container로 실행할 수 없다.
- linux container는 격리를 위해 namespace, cgroup(control group) 기술을 사용한다.
  - namespace:
    - mnt(마운트)
    - uid
    - uts(unix time sharing, hostname과 도메인 이름)
    - pid
    - network
    - ipc(inter process communication)
  - cgroup
    - 메모리, cpu, 네트워크 대역폭 등
- 이론적으로 container는 모든 리눅스 시스템에서 실행될 수 있지만 호스트의 커널을 사용하기 때문에 커널 버전에 영향을 받는다. 뿐만 아니라 특정 하드웨어 아키텍쳐용(x86, ARM 등)으로 만들어진 image는 해당 아키텍쳐에서만 실행될 수 있다.
- kubectl은 k8s control plane의 구성 요소는 API server와 요청/응답을 통해 통신한다.
- docker 기준 k8s는 po내 모든 container가 동일한 linux namespace를 공유하도록 docker를 설정한다. 파일 시스템의 경우 image에 저장되어 있어 기본적으로 완전히 분리된다.
  - 동일한 namespace: 네트워크, uts
  - 다른 namespace: uid, pid, mnt
- po 매니페스트 중 spec.containers[\*].ports[\*].containerport는 container가 노출하는 포트 정보를 명시하기만 할 뿐 다른 기능은 없다. 이를 생략한다고 해서 포트를 통해 po에 연결할 수 있는 여부에 영향을 미치지 않는다.
- container 로그는 하루 단위, 10MB 크기 기준으로 롤링(rolling)된다.
- annotation은 key-value으로 label과 유사하지만 식별 정보로 사용되지 않는다. 즉 label과 같이 selector 기능은 없다. annotation은 주로 해당 리소스에 대한 정보를 나타내는데 사용된다.
- k8s namespace는 리소스 이름의 범위를 제공한다. 뿐만 아니라 리소스를 격리하는 것외에도 특정 사용자가 지정된 리소스에 접근할 수 있도록 허용하고, 개별 사용자가 사용할 수 있는 컴퓨팅 리소스를 제한하는 데에도 사용된다. 하지만 리소간 격리는 제공하지 않는다. 즉 서로 다른 namepsace에 존재하는 리소스더라도 통신할 수 있다. 네트워크 격리는 k8s와 함께 배포되는 네트워킹 솔루션에 따라 다르다.
- 대부분의 리소스 이름은 RFC 1035 (domain name)에 지정된 규칙을 준수해야 한다. 즉, 글자, 숫자, 대시, 점을 포함할 수 있다. 하지만 몇몇 리소스는 점을 포함할 수 없다.
- k8s에서 추가하는 annotation, label을 식별하기 위해 kubernetes.io/, k8s.io/ label을 예약했다. k8가 자동으로 붙이는 label
  - ns
    - kubernetes.io/metadata.name: NamespaceDefaultLabelName feature gate가 활성화됐을 경우 추가되며 ns의 이름을 갖는다.
- owner reference는 k8s resource 간의 종속 관계를 나타낸다. k8s는 object 삭제 시 label이 아닌 owner reference를 사용해 종속 관계에 대한 cascading deletion을 수행한다. 
- container의 생명주기와 관련해 hook을 제공하며, handler를 구현함으로써 hook에 대한 이벤트를 처리할 수 있다.
  - PostStart: container와 비동기적으로 실행된다. 하지만 PreStart가 완료되지 않으면 container는 running state에 도달할 수 없다.
  - PreStop: container에 TERM 시그널이 전송되기 전에 실행된다. po의 terminationGracePeriodSeconds 설정보다 PreStop, TERM signal 전송 및 처리 시간이 더 오래걸이면 container는 비정상 종료 될 수도 있다.

## 명령어
- kubectl get [RESOURCE] --field-selector
- 

## 체크리스트
- po내 ports[*].hostPort에 사용된 port는 호스트 netstat 조회 시 보이지 않음. 하지만 type=ClusterIP svc로 expose 시 netstat에 조회됨
- svc externalIPs 설정 시, no의 IP로 svc 접근 가능
- local pv의 경우, pv 생성 시 디렉토리가 호스트 내 존재해야 함. 자동 생성되지 않음 확인