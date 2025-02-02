# [De-mystifying cluster networking for Amazon EKS worker nodes](https://aws.amazon.com/ko/blogs/containers/de-mystifying-cluster-networking-for-amazon-eks-worker-nodes/)

## EKS Cluster Architecture
amazon eks cluster는 두 개의 vpc로 구성된다. 하나는 aws가 관리하는 k8s control plane이 위치한 vpc이며, 다른 하나는 사용자가 관리하는 vpc로 k8s worker node(ec2 인스턴스)와 로드 밸런서 같은 cluster 관련 인프라가 포함된다. 모든 worker node는 control plane의 구성 요소인 kube-apiserver endpoint에 연결할 수 있어야한다. 연결을 통해 worker ndoe는 자신을 k8s control plane에 등록하고 po 실행에 대한 요청을 수신한다.

worker node는 public endpoint 또는 eks-managed elastic network interface(eni)를 통해 이뤄진다. eks-managed eni는 cluster 생성 시 명시한 subnet에 생성된다. worker node가 control plane에 연결하는 방법은 cluster에 대한 private endpoint 활성화 여부에 따라 결정된다. private endpoint가 비활성화되더라도 eks는 여전히 eks-managed eni를 provision한다. kube-apiserver는 `kubectl exec`, `kubectl logs`와 같은 명령어 대한 요청을 수행하기 eks-managed eni를 사용한다.

아래는 위 아키텍처에 대한 다이어그램이다.  
![](https://d2908q01vomqb2.cloudfront.net/fe2ef495a1152561572949784c16bf23abb28057/2020/04/10/eks_architecture.png)

amazon eks의 worker node가 control plane으로부터 명령을 받기까지의 과정은 다음과 같다.
1. ec2 인스턴스 시작: 부팅 과정에서 kubelet, k8s node agent가 실행된다.
2. no 등록: kubelet이 k8s cluster endpoint(kube-apiserver)에 연결해 no를 등록한다. vpc 외부의 public endpoint 또는 vpc 내부 private endpoint를 통해 연결을 수행한다.
3. kubelet은 API 명령을 수신하면서, 주기적으로 k8s cluster endpoint에 status와 heartbeat를 전송한다.

만약 no가 cluster endpoint에 접근하지 못하는 경우에는 control plane에 자신을 등록하지 못하며 결과적으로 po에 대한 실행, 중지를 수행할 수 없다.

## Networking modes
eks는 k8s cluster endpoint에 대한 접근을 위한 두 가지 방식을 제공한다. endpoint 접근 제어를 통해 사용자는 public 인터넷 을 통한 접근 또는 vpc를 통한 접근을 설정할 수 있다. 1) public endpoint(기본 값), 2) private endpoint, 또는 3) public + private endpoint 옵션 값을 설정할 수 있다. public endpoint를 활성화하는 경우 CIDR 접근 제어를 추가적으로 지원한다.

### Public endpoint only
eks의 기본 설정 값이다. k8s cluster의 vpc 내에서 kube-apiserver에 대한 요청은 vpc 밖으로 나간다(하지만 aws network 외부를 나가지 않는다). no가 control plane에 접근하기 위해 public ip와 internet gateway 또는 public NAT gateway가 필요하다.  
![](https://d2908q01vomqb2.cloudfront.net/fe2ef495a1152561572949784c16bf23abb28057/2020/04/10/endpoint_public.png)

### Public and Private endpoints
public endpoint, private endpoint 모두 활성화된 경우 k8s cluster의 vpc 내에서 kube-apiserver에 대한 요청은 eks-managed eni를 사용한다. 그리고 인터넷에서는 public endpoint를 통해 kube-apiserver에 접근한다.  
![](https://d2908q01vomqb2.cloudfront.net/fe2ef495a1152561572949784c16bf23abb28057/2020/04/10/endpoint_pubprivate.png)

### Private endpoint only
private endpoint만 활성화된 경우 kube-apiserver에 대한 모든 요청은 k8s cluster vpc 내부 또는 connected network를 통해서만 접근 가능하다. 인터넷을 통한 kube-apiserver에 대한 접근은 불가능하다. `kubectl` 명령어는 vpc 도는 connected network에서만 접근 가능하다.

![](https://d2908q01vomqb2.cloudfront.net/fe2ef495a1152561572949784c16bf23abb28057/2020/04/10/endpoint_private.png)

> **Sidebar: Do my worker nodes need to be in the same subnets as the ones I provided when I started the cluster?**
>
> 아니다. worker ndoe는 동일 vpc에서만 실행되면 된다.

## VPC configurations

### Public only subnets

### Public + Private subnets

### Private only subnets

### Conclusion