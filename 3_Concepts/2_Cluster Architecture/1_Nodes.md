## Management

API server에 no를 추가하기 위한 두 가지 방법이 있다:

1. no의 kubelet이 control plane에 스스로 등록
2. 사람이 직접 no object를 추가

no가 등록된 후 control plane은 새로운 no object가 유효한지 확인한다. 예를 들어 아래 JSON manifest를 통해 no를 생성할 경우:

``` json
{
  "kind": "Node",
  "apiVersion": "v1",
  "metadata": {
    "name": "10.240.79.157",
    "labels": {
      "name": "my-first-k8s-node"
    }
  }
}
```

k8s는 내부적으로 no object를 생성한다. Kubernetes checks that a kubelet has registered to the API server that matches the metadata.name field of the Node. 만약 no가 정상이면(즉, 필요한 모드 서비스가 실행 중) po를 실행할 자격이 있다. 정상이 아니라면 정상이 되기 전까지 해당 no는 클러스터와 관련된 행동에서 제외된다.

**Note**: k8s는 유효하지 않은 no의 object를 보존하면서 정상이 될때까지 게속 체크한다. health check를 멈추기 위해 no object를 직접 또는 controller가 삭제해야 한다.

no object의 이름은 DNS subdomain name 규칙을 따라야한다.

### Node name uniqeness
두 개의 no가 동시에 동일한 이름을 가질 수 없다. k8s는 동일한 이름의 resource에 대해 동일한 object로 생각한다. 동일한 이름을 갖는 no의 경우 동일한 state(네트워크, root disk 내용), label 등의 정보를 갖는다고 생각한다. 이로 인해 이름을 변경하지 않고 인스턴스가 수정되면 불일치가 발생할 수 있다. no를 교체하거나 업데이트해야 하는 경우 기존 no object를 먼저 API에서 제거하고 업데이투 후 다시 추가해야 한다.

### Self-registration of Nodes
kubelet에 --register-node flag를 true(기본 값)으로 설정하면 kubelet은 API server에 스스로 등록한다. 대부분의 경우 선호하는 방식이다.

스스로 등록할 경우 kubelet에 아래 옵션을 설정해야 한다:

- --kubeconfig: API server에 인증하기 위해 사용되는 credentials의 경로
- --cloud-provider: metadata를 읽기 위해 cloud provier와 통신하는 방법
- --register-node: API server에 스스로 등록할지 여부
- --register-with-taints: no의 taints 목록(\<key\>=\<value\>:\<effect\>를 ,로 구분)
- --node-ip: no의 IP 주소
- --node-labels: no의 label (NodeRestriction admission plugin에 의해 강제되는 label 규칙도 있다)
- --node-status-update-frequency: kubelet이 no의 상태를 API server에 보고하는 빈도

 Node authorization mode, NodeRestriction admission plugin가 활성화된 경우, kubelet은 자체 no의 resource만 생성/수정할 수 있는 권한이 있다. 

**Note**: no의 설정 변경이 필요한 경우 API server에 no를 다시 등록하는 것이 권장 방법이다. 예를 들어 kubelet이 새로운 --node-labels 옵션을 사용해 재시작 되지만 동일한 no의 이름이 사용되는 경우 label이 no 등록에 설정되어있으므로 변경 사항이 적용되지 않는다.

kubelet 재시작 시 no의 설정이 변경되는 경우 해당 no에 스케줄링된 po는 오작동하거나 문제를 일으킬 수 있다. For example, already running Pod may be tainted against the new labels assigned to the Node, while other Pods, that are incompatible with that Pod will be scheduled based on this new label. Node re-registration ensures all Pods will be drained and properly re-scheduled.

### Manual Node administration
kubelet을 사용해 no object를 생성, 수정할 수 있다.

직접 no object를 생성하기 위해 --register-node=false flag를 사용한다.

--register-node flag와 상관없이 no obejct를 수정할 수 있다. 예를 들어 label 수정할 수 있다.

no의 label을 po의 label selector와 같이 사용해 스케줄링을 제어할 수 있다.

no를 스케줄링 불가능하도록 만들면 scheduler는 해당 no에 새로운 po를 스케줄링할 수 없지만 기존 po에는 영향을 미치지 않는다. 이는 no의 재부팅, 기타 유지 보수 준비 단계를 위해 유용하다.

no를 스케줄링 불가하게 하기 위해 아래 명령어를 사용할 수 있다:

``` bash
kubectl cordon $NODENAME
```

자세한 내용은 [Safely Drain a Node](https://v1-24.docs.kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/) 페이지를 참고한다.

**Note**: ds의 일부인 po는 스케줄링 불가한 no에서도 실행될 수 있다. ds는 일반적으로 workload 애플리케이션이 제거되더라도 no에서 실행되어야 하는 no lacal 서비스다.

## Node status