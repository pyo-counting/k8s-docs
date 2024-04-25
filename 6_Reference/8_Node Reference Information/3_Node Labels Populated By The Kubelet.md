k8s no는 미리 정의된 표준 label 집합을 갖는다.

물론 kubelet 설정, k8s API를 사용해 no에 직접 label을 추가할 수도 있다.

## Preset labels
k8s가 기본적으로 no에 설정하는 표준 label은 다음과 같다(k8s 구성 요소 중 kubelet이 설정).
- `kubernetes.io/arch`
- `kubernetes.io/hostname`
- `kubernetes.io/os`
- `node.kubernetes.io/instance-type` (if known to the kubelet – Kubernetes may not have this information to set the label)
- `topology.kubernetes.io/region` (if known to the kubelet – Kubernetes may not have this information to set the label)
- `topology.kubernetes.io/zone` (if known to the kubelet – Kubernetes may not have this information to set the label)

> **Note**:  
> label 값은 cloud provider마다 다를 수 있으며 보장되지 않는다. 예를 들어, `kubernetes.io/hostname` label의 값은 일부 환경에서 no 이름과 동일할 수 있고 다른 환경에서는 다른 값을 가질 수도 있다.