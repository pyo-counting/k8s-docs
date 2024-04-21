[kube-scheduler](https://kubernetes.io/docs/concepts/scheduling-eviction/kube-scheduler/#kube-scheduler)는 k8s의 기본 scheduler다. kube-scheduler는 cluster 내에서 po를 no에 배치하는 역할을 수행한다.

cluster 내에서 po의 스케줄링 요구 사항을 충족하는 no를 해당 po를 feasible no라고 부른다. kube-scheduler는 적합한 no를 찾은 다음, no들에 대한 점수를 매기는 일련의 함수를 실행하고 가장 높은 점수를 받은 feasible no를 선택해 po를 할당한다. 그런 다음 scheduler는 이러한 결정을 binding이라는 프로세스에서 kube-apiserver에 통지한다.

이 페이지에서는 대규모 k8s cluster의 성능 튜닝에 대해 설명한다.

대규모 cluster에서는 kube-scheduler를 튜닝해 스케줄링 결과를 지연시간(새 po가 no에 할당되는 데 걸리는 시간)과 정확성(kube-scheduler가 적절한 no에 po를 배치하는 지 여부) 사이에서 균형을 맞출 수 있다.

튜닝은 kube-scheduler의 `percentageOfNodesToScore`을 통해 설정한다. 이 설정은 cluster 내에서 no를 스케줄링하기 위한 임계ㄱ밧을 결정한다.

### Setting the threshold
## Node scoring threshold
### Default threshold
## Example
## Tuning percentageOfNodesToScore
## How the scheduler iterates over Nodes