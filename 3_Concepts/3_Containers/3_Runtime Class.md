RuntimeClass resource와 runtime 선택 메커니즘에 대해 설명한다.

RuntimeClass는 container runtime config을 선택하는 기능이다. container runtime config는 po의 container를 실행하는 데 사용된다.

## Motivation
서로 다른 po간에 다른 RuntimeClass를 사용할 수 있다.

## Setup
1. Configure the CRI implementation on nodes
RuntimeClass를 통해 설정 가능한 기능은 CRI(Container Runtime Interface) 구현에 따라 다르다.

**Note**: RuntimeClass assumes a homogeneous node configuration across the cluster by default (which means that all nodes are configured the same way with respect to container runtimes). To support heterogeneous node configurations, see Scheduling below.

no에서 CRI에 대한 설정은 handler 이름을 가지며 이는 RuntimeClass에서 참조한다.

2. Create the corresponding RuntimeClass resources
RuntimeClass resource는 2개의 중요 필드만 갖는다: `metadata.name`과 `handler`.

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

**Note**: RuntimeClass write 작업(create/update/patch/delete)은 클러스터 관리자에 제한하는 것을 권장한다. 이는 기본설정이다.

## Usage
po .spec.runtimeClassName을 통해 사용할 수 있다.

kubelet은 명시된 RuntimeClass를 사용해 po를 실행한다. 만약 명시된 RuntimeClass가 없거나, CRI가 상응하는 핸들러를 실행할 수 없는 경우, po는 Failed terminal phase에 진입한다. 관련된 에러 메시지는 event를 확인한다.

만약 명시된 runtimeClassName가 없다면 기본 RuntimeHandler가 사용되며, RuntimeClass 기능이 비활성화되었을 때와 동일하게 동작한다.

### CRI Configuration
#### dockershim
dockershim을 사용하는 경우 RuntimeClass는 runtime handler를 docker로 고정한다. dockershim은 사용자 정의 runtime handler를 지원하지 않는다.

#### containerd

#### CRI-O

## Scheduling
RuntimeClass에 scheduling 필드를 지정하면, 이 RuntimeClass로 실행되는 po가 이를 지원하는 no로 스케줄링되도록 제약 조건을 설정할 수 있다. scheduling이 설정되지 않은 경우 이 RuntimeClass는 모든 no에서 지원되는 것으로 간주된다.

po가 지정한 RUntimeClass를 지원하는 no에 스케줄링되는 것을 보장하려면, 해당 no들은 runtimeClass.scheduling.nodeSelector 필드에서 선택되는 공통 label을 가져야한다. RuntiemClass의 nodeSelector는 po의 nodeSelector와 API Server의 admission에서 병합되어서 실질적으로 각각에 의해 필터된 no가 선택된다. 충돌이 있는 경우, po는 거부된다.

If the supported nodes are tainted to prevent other RuntimeClass pods from running on the node, you can add tolerations to the RuntimeClass. As with the nodeSelector, the tolerations are merged with the pod's tolerations in admission, effectively taking the union of the set of nodes tolerated by each.

### Pod Overhead