ingress 리소스가 정상적으로 동작하기 위해서는 클러스터 내에 관련 ingress controller가 실행 중이어야 한다.

kube-controller-manager에 의해 동작하는 다른 controller와 다르게 ingress controller는 클러스터에서 자동으로 실행되지 않는다. 아래에서는 사용자 클러스터 환경에 맞는 ingress controller 선택을 위한 내용을 제공한다.

k8s에서는 AWS, GCE, nginx ingress controller를 프로젝트로 관리한다.

## Additional controllers

## Using multiple Ingress controllers
k8s ingress class resource를 사용해 여러 종류의 ingress controller를 배포할 수 있다. ingress를 생성할 때 .spec.ingressClassName 필드에 ingress class 이름을 명시해야 한다. .spec.ingressClassName 필드는 이전 annotation을 사용하던 방식을 대체한다.

클러스터 내 1개의 ingress class만 있고 기본 값일 경우 .spec.ingressClassName을 명시하지 않으면 기본 ingress class를 사용한다. ingress class에 ingressclass.kubernetes.io/is-default-class=true annotation을 설정해 기본으로 설정이 가능하다.

모든 ingress controller가 위와 같은 규격을 충족해야 하지만, 각 ingress controller마다 동작이 상이할 수 있다.

**Note**: ingress controller의 documentation을 살펴보고 이러한 차이점과 주의 사항을 이해해야 한다.