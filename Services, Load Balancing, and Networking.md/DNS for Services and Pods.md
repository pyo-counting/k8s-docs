## DNS for Services and Pods
k8s는 svc, po를 위한 DNS record를 생성한다.

### Intruduction
k8s DNS는 클러스터의 DNS po, svc를 관리하며 개별 container가 DNS 이름을 해석할 때 DNS svc의 IP를 사용하도록 kubelets를 구성(/etc/resolv.conf 파일 참고)한다.

클러스터 내의 모든 svc(DNS 서버 자신도 포함)에는 DNS 이름이 할당된다. 기본적으로 클라이언트 po의 DNS search directive 목록에는 po의 ns와 클러스터 default domain을 포함한다.

#### Namespaces of Services
DNS 쿼리는 po의 ns에 따라 다른 결과를 반환한다. ns를 명시하지 않은 DNS 쿼리는 po의 ns에 제한된다. 다른 ns의 svc에 대한 접근을 위해 DNS 쿼리에 ns를 명시해야 한다.

예를 들어 test ns에 po가 있고, prod ns에 data svc가 있다.

이 때 po에서 data DNS 쿼리에 대한 반환 값은 없다. 왜냐하면 po의 ns가 test기 때문이다.

하지만 data.prod DNS 쿼리에 대한 반환은 해당 svc의 IP가 포함된다.

DNS 쿼리는 po의 /etc/resolv.conf를 사용해 확장될 수 있다. kubelet은 각 po에서 해당 파일을 설정한다. 예를 들어 data에 대한 DNS 쿼리는 data.test.svc.cluster.local 쿼리로 확장된다. 이 DNS 쿼리 요청은 search directive의 값을 통해 확장된 것이다. 아래는 po 내 resolv.conf 파일 예시다.

```
nameserver 10.32.0.10
search <namespace>.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

#### DNS Records
DNS recors를 갖는 k8s object는 아래와 같다.

1. svc
2. po

