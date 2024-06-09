kube-apiserver, kube-proxy와 같은 일부 [Kubernetes components](https://kubernetes.io/docs/concepts/overview/components/)는 cluster 내에서 [container image](https://kubernetes.io/releases/download/#container-images)를 사용해 container로 실행할 수 있다.

가능한 한 k8s 구성 요소를 container image로 실행하고 k8s가 해당 구성 요소를 관리하도록 하는 것을 권장한다. 하지만 container 실행을 담당하는 kubelet은 container로 실행할 수 없다.

직접 k8s를 관리하지 않기 위해 [certified platform](https://kubernetes.io/docs/setup/production-environment/turnkey-solutions/)을 포함한 managed service를 사용할 수 있다. 또한 광범위한 클라우드 및 베어메탈 환경에 걸쳐 표준화된 맞춤형 솔루션이 있습니다.

## Learning environment

## Production environment