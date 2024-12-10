po는 생성되면 일단 스케줄링을 위한 준비가 완료 된것으로 간주된다. 그렇기 때문에 k8s scheduler는 즉시 pending pod를 배치할 no를 찾기 위해 노력한다. 그러나 일부 po는 장시간 "miss-essential-resources" 상태로 유지될 수도 있다. 이러한 po로 인해 원하지 않는 동작이 발생할 수 있다. 예를 들어 Cluster AutoScaler가 동작할 수도 있다.

po의 `.spec.schedulingGate` 필드를 명시/삭제함으로써 po가 언제 scheduling 될 준비가 됐는지 제어할 수 있다.

## Configuring Pod schedulingGates
`.spec.schedulingGate` 필드에 문자열 목록을 명시할 수 있으며 각 문자열은 po가 스케줄링 될 준비가 됐다고 간주되기 전에 충족해야 하는 조건을 나타내는데 사용한다. 이 필드는 po가 생성될 때만 명시할 수 있다. 즉 po가 생성(schedulding이 아닌 kube-apiserver에 등록)된 후에는 각 필드를 임의의 순서로 제거할 수 있지만, 추가로 새로운 목록을 추가할 수는 없다.

![](https://kubernetes.io/docs/images/podSchedulingGates.svg)

## Usage example
## Observability
## Mutable Pod scheduling directives