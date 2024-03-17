## Going back in time
kubernetes는 선언적 설정, 자동화를 모두 용이하게하는 containerized workload와 서비스를 관리하기 위한 휴대성, 확장성이 있는 오픈 소스 플랫폼이다. kubernetes는 빠르게 성장하는 대규모 생테계를 갖고있다.

kubernetes 이름은 그리스어에서 유래됐으며 조타수 또는 조종사를 의미한다. 약어 k8s는 "k"와 "s" 사이에 8글자가 있음을 의미한다. Google은 2014년에 kubernetes를 오픈 소스로 공개했다. kubernetes는 Google의 15년 이상 경험과 커뮤니티의 최고 수준의 아이디어 및 관행을 갖고있다.

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
    - dev, ops 문제 분리: reate application container images at build/release time rather than deployment time, thereby decoupling applications from infrastructure.
    - 관측 가능성: 운영 체제 수준의 정보와 metric을 표면화할 뿐만 아니라 애플리케이션 health, 기타 신호도 확인할 수 있다.
    - 개발, 테스트, 운영 환경 전반에 걸친 일관성: 클라우드에서와 마찬가지로 노트북에서도 동일하게 실행된다.
    - 클라우드, 운영 체제 배포 휴대성: Ubuntu, RHEL, CoreOS, 주요 public cloud 등 모든 곳에서 실행 가능하다.
    - 애플리케이션 중심 관리: 가상 하드웨어에서 운영 체제를 실행하는 것에서 논리 리소스를 사용하는 운영 체제에서 애플리케이션을 실행해 추상화 레벨을 높인다.
    - 느슨하게 결합되고 분산되고 탄력적인 마이크로 서비스: 애플리케이션은 더 작고 독립적인 조각으로 나뉘며 동적으로 배포 및 관리할 수 있다. 단일 대규모 단일 목적 시스템에서 실행되는 단일 스택이 아니다.
    - 리소스 격리: 예측 가능한 애플리케이션 성능
    - 리소스 활용도: 높은 효율성과 밀도

## Why you need Kubernetes and what it can do

## What Kubernetes is not