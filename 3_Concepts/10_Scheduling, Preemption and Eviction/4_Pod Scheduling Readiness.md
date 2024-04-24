po는 생성되면 일단 스케줄링을 위한 준비가 된것으로 간주된다. k8s scheduler는 pending pod를 배치할 no를 찾기 위해 노력한다. 그러나 일부 po는 장시간 "miss-essential-resources" 상태로 유지될 수도 있다. 이러한 po로 인해 원하지 않는 동작이 발생할 수 있다. 예를 들어 Cluster AutoScaler가 동작할 수도 있다.

po의 `.spec.schedulingGate` 필드를 명시/삭제함으로써 po가 언제 scheduling 될 준비가 됐는지 제어할 수 있다.

## Configuring Pod schedulingGates
![](https://kubernetes.io/docs/images/podSchedulingGates.svg)

## Usage example
## Observability
## Mutable Pod scheduling directives