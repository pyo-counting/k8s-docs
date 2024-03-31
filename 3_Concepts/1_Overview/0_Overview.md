kubernetes는 선언적 설정, 자동화를 모두 용이하게하는 containerized workload와 서비스를 관리하기 위한 휴대성, 확장성이 있는 오픈 소스 플랫폼이다. kubernetes는 빠르게 성장하는 대규모 생테계를 갖고있다.

kubernetes 이름은 그리스어에서 유래됐으며 조타수 또는 조종사를 의미한다. 약어 k8s는 "k"와 "s" 사이에 8글자가 있음을 의미한다. Google은 2014년에 kubernetes를 오픈 소스로 공개했다. kubernetes는 [Google의 15년 이상 경험](https://kubernetes.io/blog/2015/04/borg-predecessor-to-kubernetes/)과 커뮤니티의 최고 수준의 아이디어 및 관행을 갖고있다.

## Going back in time
k8s가 유용한 이유를 살펴본다.
![img](https://kubernetes.io/images/docs/Container_Evolution.svg)

- Traditional deployment era: 초기에 조직은 물리적 서버에서 애플리케이션을 실행했다. 물리적 서버에서는 애플리케이션에 대한 리소스 경계를 정의할 방법이 없었기 때문에 리소스 할당 문제가 발생했습니다. 예를 들어 물리적 서버에서 여러 애플리케이션이 실행되면 한 애플리케이션이 대부분의 리소스를 차지할 수도 있으며 결과적으로 다른 애플리케이션이 성능이 떨어지는 경우가 발생할 수 있다. 이를 위한 해결책은 각 애플리케이션을 다른 물리적 서버에서 실행하는 것이였다. 하지만 이러한 스케일링은 리소스에 대한 충분한 활용은 아니였으며 조직이 수 많은 물리적 서버를 유지 관리하는 데 비용이 많이 들었다.
- Virtualized deployment era: 해결책으로 가상화를 도입했다. 가상화는 하나의 물리적 서버 CPU에서 여러 개의 VM(가상 머신)을 실행할 수 있도록 한다. 가상화를 통해 애플리케이션을 VM 단위로 분리할 수 있으며 한 애플리케이션의 정보를 다른 애플리케이션이 자유롭게 접근할 수 없기 때문에 보안을 제공할 수 있다.

    가상화를 통해 물리적 서버의 리소스를 더 잘 활용할 수 있고 애플리케이션을 쉽게 추가하거나 업데이트할 수 있고 하드웨어 비용을 절감할 수 있으며 훨씬 더 많은 것을 할 수 있기 때문에 확장성이 더 좋다. 가상화를 사용하면 물리적 리소스를 일회용 가상 시스템의 클러스터로 나타낼 수 있다.

    각 VM은 가상화된 하드웨어 상위 레벨에서 자체 운영 체제를 포함한 모든 구성 요소를 실행하는 전체 시스템이다.
- Container deployment era: container는 VM과 비슷하지만 애플리케이션 간 운영 체제를 공유하기 위해 격리 기능이 완화된다. 따라서 container는 가볍다. VM과 유사하게 container에는 자체 파일 시스템, CPU 공유, 메모리, 프로세스 공간 등이 있다. container는 하위 레벨의 인프라와 분리되어 있으므로 클라우드와 OS 배포판에 상관 없이 제공할 수 있다.
    
    container는 다음과 같은 추가적인 이점을 제공하기 때문에 인기 있다.
    - 빠른 애플리케이션 생성, 배포: VM image 사용에 비해 container image 생성의 용이성과 효율성이 향상됐다.
    - 지속적인 개발, 통합, 배포: (image immutability으로 인해) 빠르고 효율적인 롤백을 통한 신뢰성과 함께 빈번한 conatiner image 빌드, 배포를 제공한다.
    - dev, ops 문제 분리: create application container images at build/release time rather than deployment time, thereby decoupling applications from infrastructure.
    - 관측 가능성: 운영 체제 수준의 정보와 metric을 표면화할 뿐만 아니라 애플리케이션 health, 기타 신호도 확인할 수 있다.
    - 개발, 테스트, 운영 환경 전반에 걸친 일관성: 클라우드에서와 마찬가지로 노트북에서도 동일하게 실행된다.
    - 클라우드, 운영 체제 배포 휴대성: Ubuntu, RHEL, CoreOS, 주요 public cloud 등 모든 곳에서 실행 가능하다.
    - 애플리케이션 중심 관리: 가상 하드웨어에서 운영 체제를 실행하는 것에서 논리 리소스를 사용하는 운영 체제에서 애플리케이션을 실행해 추상화 레벨을 높인다.
    - 느슨하게 결합되고 분산되고 탄력적인 마이크로 서비스: 애플리케이션은 더 작고 독립적인 조각으로 나뉘며 동적으로 배포 및 관리할 수 있다. 단일 대규모 단일 목적 시스템에서 실행되는 단일 스택이 아니다.
    - 리소스 격리: 예측 가능한 애플리케이션 성능
    - 리소스 활용도: 높은 효율성과 밀도

## Why you need Kubernetes and what it can do
container는 애플리케이션을 실행하는 좋은 방법이다. 운영 환경에서는 애플리케이션을 실행하는 container를 관리하고 downtime이 없는지 확인해야 한다. 예를 들어 container가 다운되면 다른 container를 다시 시작해야 한다면 이를 시스템에 의해 처리한다면 더 쉬울 것이다.

이러한 이유로 k8s가 필요하다. k8s는 분산 시스템을 탄력적으로 실행하기 위한 프레임워크를 제공한다. 애플리케이션의 스케일링과 failover를 처리하고 배포 패턴 등을 제공한다. 예를 들어, k8s는 canary 배포를 쉽게 관리할 수 있다.

k8s는 다음을 제공한다:
- service discovery와 로드 밸런싱: k8s는 DNS 이름을 사용하거나 자체 IP 주소를 사용하여 container를 노출할 수 있다. container에 대한 트래픽이 많으면 k8s는 트래픽을 분산하기 때문에 배포가 안정적으로 이루어질 수 있다.
- storage orchestration: k8s를 사용하면 로컬 storage, public cloud provider 등과 같이 원하는 storage 시스템을 마운트할 수 있다
- 자동화된 rollout과 rollback: k8s를 사용하여 배포된 container의 desired state를 서술할 수 있으며 current state를 desired statge로 설정한 속도에 따라 변경할 수 있다. 예를 들어 k8s를 자동화해서 배포용 새 container를 만들고, 기존 container를 제거하고, 모든 리소스를 새 container에 적용할 수 있다.
- 자동화된 bin packing: 사용자는 container화된 작업을 실행하는데 사용할 수 있는 k8s 클러스터 no를 제공한다. 각 container가 필요로 하는 CPU와 메모리(RAM)를 설정할 수 있으며 k8s는 container를 no에 맞추어서 리소스를 가장 잘 사용할 수 있도록 해준다.
- 자동화된 복구(self-healing): k8s는 실패한 container를 다시 시작하고, container를 교체하며, 사용자 정의 health check에 응답하지 않는 container를 재시작한다. container가 트래픽을 처리할 준비가 되기 전까지 노출하지 않는다.
- secret, configuration 관리: k8s를 사용하면 패스워드, OAuth 토큰, SSH 키와 같은 중요한 정보를 저장하고 관리할 수 있다. 사용자는 container image를 다시 빌드할 필요가 없다.
- baatch 실행: k8s는 batch, ci workload를 관리할 수 있으며, 장애가 발생한 container를 교체할 수 있다.
- horizontal scaling: 명령어, ui, 또는 cpu 사용량에 따라 애플리케이션을 스케일링할 수 있다.
- ipv4, ipv6 dualstack: po, svc에 ipv4, ipv6 주소를 할당할 수 있다.
- designed for extensibility:

## What Kubernetes is not
k8s는 모든 기능을 포함하는 전통적인 Platform as a Service(PaaS)가 아니다. k8s는 하드웨어 수준보다는 container 수준에서 운영되기 때문에 PaaS가 일반적으로 제공하는 배포, 스케일링, 로드 밸런싱과 같은 기능을 제공하며 사용자가 로깅, 모니터링, 알림 솔루션을 통합할 수 있다. 하지만, k8s는 모놀리식(monolithic)이 아니어서, 이런 기본 솔루션이 선택적이며 추가나 제거가 용이하다. k8s는 개발자 플랫폼을 만드는 구성 요소를 제공하지만, 필요한 경우 사용자의 선택권과 유연성을 지켜준다.

k8s는:
- 지원하는 애플리케이션의 유형을 제약하지 않는다. k8s는 상태 유지가 필요 없는(stateless) workload, 상태 유지가 필요한(stateful) 워크로드, 데이터 처리를 위한 workload를 포함한 다양한 worklaod를 지원하는 것을 목표로 한다. 애플리케이션이 container에서 구동될 수 있다면, k8s에서도 잘 동작할 것이다.
- 소스 코드를 배포하지 않으며 애플리케이션을 빌드하지 않는다. 지속적인 통합과 전달과 배포, 곧 CI/CD 워크플로우는 조직 문화와 취향에 따를 뿐만 아니라 기술적인 요구사항으로 결정된다.
- 애플리케이션 레벨의 서비스를 제공하지 않는다. 애플리케이션 레벨의 서비스에는 미들웨어(예, 메시지 버스), 데이터 처리 프레임워크(예, Spark), 데이터베이스(예, MySQL), 캐시 또는 클러스터 스토리지 시스템(예, Ceph) 등이 있다. 이런 컴포넌트는 k8s 상에서 구동될 수 있고, k8s 상에서 구동 중인 애플리케이션이 [Open Service Broker](https://www.openservicebrokerapi.org/)와 같은 이식 가능한 메커니즘을 통해 접근할 수도 있다.
- 로깅, 모니터링 또는 알람 솔루션을 포함하지 않는다. proof of concept을 위한 일부 integration, 메트릭을 수집하고 노출하는 메커니즘을 제공한다.
- 기본 설정 언어/시스템(예, Jsonnet)을 제공하거나 요구하지 않는다. declarative specification의 임의적인 형식을 목적으로 하는 declarative API를 제공한다.
- Does not provide nor adopt any comprehensive machine configuration, maintenance, management, or self-healing systems.
- 추가로 k8s는 단순한 orchestration 시스템이 아니다. 사실, k8s는 orchestration의 필요성을 없애준다. orchestration의 기술적인 정의는 A -> B -> C 처럼 순서가 정의된 워크플로우를 수행하는 것이다. 반면에, k8s는 독립적이고 조합 가능한 제어 프로세스들로 구성되어 있다. 이 프로세스는 지속적으로 현재 상태를 입력받은 desired state로 나아가도록 한다. A에서 C로 어떻게 갔는지는 상관이 없다. 중앙화된 제어도 필요치 않다. 이로써 시스템이 보다 더 사용하기 쉬워지고, 강력해지며, 견고하고, 회복력을 갖추게 되며, 확장 가능해진다.
