RuntimeClass resource와 runtime 선택 메커니즘에 대해 설명한다.

RuntimeClass는 container runtime config을 선택하는 기능이다. container runtime config는 po의 container를 실행하는 데 사용된다.

## Motivation
서로 다른 po간에 다른 RuntimeClass를 사용해 성능과 보안의 균형을 유지할 수 있다. 예를 들어 workload의 일부가 높은 수준의 정보 보안 유지가 필요한 경우 하드웨어 가상화를 사용하는 container runtime에서 po가 실행되도록 할 수 있다. 이를 통해 추가 overhead를 감수하고 대체 runtime을 추가로 격리할 수 있는 이점을 얻을 수 있다.

또한 RuntimeClass를 사용해 동일한 container runtime을 사용하면서 다론 설정을 이용해 po들을 실행할 수 있다.

## Setup
### 1. Configure the CRI implementation on nodes
RuntimeClass를 통해 이용 가능한 설정은 CRI(Container Runtime Interface)의 구현에 따라 다르다.

> **Note**:  
> RuntimeClass 기본적으로 cluster 전체에서 동일한 종류의 no 구성을 가정한다(container runtime과 관련해 모든 no가 동일하게 구성). 다른 종류의 no 구성은 아래 Scheduling을 참고한다.

no에서 container runtime는 RuntimeClass에서 참조하기 위해 handler에 대응하는 설정을 갖는다. handler는 [DNS label name](https://kubernetes.io/docs/concepts/overview/working-with-objects/names/#dns-label-names)을 충족해야 한다.

### 2. Create the corresponding RuntimeClass resources
각 handler에 대해 RuntimeClass object를 생성한다. 현재 RuntimeClass resource는 2개의 중요 필드만 갖는다: `metadata.name`과 `handler`.
``` yaml
# RuntimeClass is defined in the node.k8s.io API group
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  # The name the RuntimeClass will be referenced by.
  # RuntimeClass is a non-namespaced resource.
  name: myclass 
# The name of the corresponding CRI configuration
handler: myconfiguration
```
RUntimeClass object의 이름은 [DNS subdomain name](https://kubernetes.io/docs/concepts/overview/working-with-objects/names/#dns-subdomain-names) 규칙을 따라야 한다.

> **Note**:  
> RuntimeClass object에 대한 write 작업(create/update/patch/delete)은 클러스터 관리자에 제한하는 것을 권장한다. 이는 기본설정이다.

## Usage
RuntimeClass object를 설정했다면, 사용하는 것은 아주 쉽다. po `.spec.runtimeClassName`을 통해 사용할 수 있다.
``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  runtimeClassName: myclass
  # ...
```

kubelet은 명시된 RuntimeClass를 사용해 po를 실행한다. 만약 명시된 RuntimeClass가 없거나, CRI가 대응하는 handler를 실행할 수 없는 경우, po는 Failed terminal phase에 진입한다. 관련된 에러 메시지는 event를 확인한다.

만약 runtimeClassName을 설정하지 않으면 기본 RuntimeHandler가 사용되며 이는 RuntimeClass 기능이 비활성화되었을 때와 동일하게 동작한다.

### CRI Configuration
#### containerd
containerd의 설정 파일인 /etc/containerd/config.toml을 통해 runtime handler가 설정된다. runtime selection 내에 유효한 handler를 설정한다.
```
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.${HANDLER_NAME}]
```

자세한 내용은 [configuration documentation](https://github.com/containerd/containerd/blob/main/docs/cri/config.md)을 참고한다.

#### CRI-O

## Scheduling
RuntimeClass에 scheduling 필드를 지정하면, 이 RuntimeClass로 실행되는 po가 이를 지원하는 no로 스케줄링되도록 제약 조건을 설정할 수 있다. scheduling이 설정되지 않은 경우 이 RuntimeClass는 모든 no에서 지원되는 것으로 간주된다.

po가 지정한 RuntimeClass를 지원하는 no에 스케줄링되는 것을 보장하려면, 해당 no들은 runtimeClass.scheduling.nodeSelector 필드에서 선택되는 공통 label을 가져야한다. RuntiemClass의 nodeSelector는 po의 nodeSelector와 API Server의 admission에서 병합되어서 실질적으로 각각에 의해 필터된 no가 선택된다. 충돌이 있는 경우, po는 거부된다.

If the supported nodes are tainted to prevent other RuntimeClass pods from running on the node, you can add tolerations to the RuntimeClass. As with the nodeSelector, the tolerations are merged with the pod's tolerations in admission, effectively taking the union of the set of nodes tolerated by each.

### Pod Overhead
po 실행과 관련된 overhead resource를 지정할 수 있다. overhead를 선언하면 cluster(scheduler 포함)가 po, resource에 대한 결정을 내릴 때 overhead를 고려할 수 있다.

po overhead는 RuntimeClass의 overhead 필드를 통해 정의된다. 이 필드를 사용해 해당 RuntimeClass를 사용하는 실행 po의 overhead를 지정하고 이러한 overhead가 k8s에서 설명되도록 할 수 있다.

관련된 내용은 [Pod Overhead](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-overhead/)을 참고한다.