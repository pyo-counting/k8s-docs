k8s에서 스케줄링은 kubelet이 po를 실행할 수 있는 적합한 no를 찾는 것을 말한다. preemption은 우선 순위가 높은 po가 no에 스케줄링 될 수 있도록 우선 순위가 낮은 po를 종료하는 것을 말한다. eviction은 no에서 po를 종료하는 것을 말한다.

## Scheduling

## Pod Disruption
[pod disruption](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/)은 no의 po가 자발적 또는 비자발적으로 종료되는 프로세스다.

voluntary disruption은 사용자에 의해 수행된다. involuntary disruption은 의도적이지 않은 것으로 no의 리소스 부족과 같은 불가피한 문제 또는 실수로 인한 삭제로 발생할 수 있다.