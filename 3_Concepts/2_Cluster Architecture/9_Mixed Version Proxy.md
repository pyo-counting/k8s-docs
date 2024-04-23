k8s 1.30은 resource 요청을 kube-apiserver가 다른 peer kube-apiserver로 proxy 처리할 수 있는 alpha 기능이 포함된다. 이 기능은 하나의 cluster에서 서로 다른 버전의 k8s에 대한 kube-apiserver가 있을 때(예를 들어, k8s의 새로운 release가 장기간 rollout 하는 동안) 유용하다.

이를 통해 cluster 관리자는 resource 요청(업그레이드 중에 수행)을 올바른 kube-apiserver로 direct함으로써 안전하게 업그레이드할 수 있는 가용성이 높은 cluster를 구성할 수 있다. 이 proxy를 사용하면 업그레이드 프로세스에서 발생할 수 있는 예기치 않은 404 Not Found 오류를 방지할 수 있다.

이 메커니즘을 mixed version proxy라고 부른다.

## Enabling the Mixed Version Proxy
### Proxy transport and authentication between API servers
### Configuration for peer API server connectivity
## Mixed version proxying
### How it works under the hood