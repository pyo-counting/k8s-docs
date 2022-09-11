k8s에서 스케줄링은 kubelet이 po를 실행할 수 있도록 po가 no에 적합한지 확인하는 것을 말한다.

## Scheduling overview
scheduler는 no가 할당되지 않은 새로 생성된 po를 감시(watch)한다. scheduler가 발견한 모든 po에 대해 scheduler는 해당 po가 실행될 최상의 no를 찾는 책임을 갖는다. scheduler는 아래 설명할 스케줄링 원칙을 고려하여 이 배치를 결정헌다.

## kube-scheduler
kube-scheduler는 k8s의 기본 scheduler이며 control plane의 일부로 실행된다. 필요할 경우 자체 스케줄링 컴포넌트를 만들고 대신 사용할 수 있다.

새로 생성된 모든 po 또는 스케줄링 되지 않은 po에 대해 kube-scheduler는 실행할 최적의 no를 선택한다. 그러나 po의 모든 container에는 리소스에 대한 요구사항이 다르며 모든 po에도 요구사항이 다르다. 따라서 no들은 스케줄링 요구사항에 따라 필터링 되어야 한다.

클러스터에서 po에 대한 스케줄링 요구사항을 충족하는 no를 실행 가능한(feasible) no라고 한다. 적합한 no가 없으면 scheduler가 배치할 수 있을 때까지 po가 스케줄 되지 않은 상태로 유지된다.

scheduler는 po가 실행 가능한(feasible) no를 찾은 다음 실행 가능한(feasible) no의 점수를 측정하는 함수들의 집합을 실행하고 실행 가능한(feasible) no 중에서 가장 높은 점수를 가진 no를 선택하여 po를 실행한다. 그런 다음 scheduler는 바인딩(binding)이라고 불리는 프로세스에서 이 결정에 대해 API 서버에 알린다.

스케줄링 결정을 위해 고려해야 할 요소에는 개별 및 집단 리소스 요구사항, 하드웨어 / 소프트웨어 / 정책 제한 조건, affinity 및 anti-affinity, 데이터 지역성(data locality), 워크로드 간 간섭 등이 포함된다.

### Node selection in kube-scheduler
kube-scheduler는 2단계 작업을 통해 po를 위한 no를 선택한다.

1. 필터링(Filtering)
2. 스코어링(Scoring)

필터링 단계는 파드를 스케줄링 할 수 있는 no 집합을 찾는다. 예를 들어 PodFitsResources 필터는 후보 no가 po의 특정 리소스 요청을 충족시키기에 충분한 가용 리소스가 있는지 확인한다. 이 단계 이후 no 목록에는 적합한 no들이 포함된다. 하나 이상의 no가 포함된 경우가 종종 있을 것이다. 목록이 비어 있으면 해당 po는 (아직) 스케줄링 될 수 없다.

스코어링 단계에서 scheduler는 목록에 남아있는 no의 순위를 결정해 가장 적합한 po 배치를 선택한다. scheduler는 사용 중인 스코어링 규칙에 따라 이 점수를 기준으로 필터링 단계에서 통과된 각 no에 대해 점수를 평가한다.

마지막으로 kube-scheduler는 po를 순위가 가장 높은 no에 할당한다. 점수가 같은 no가 두 개 이상인 경우 kube-scheduler는 이들 중 하나를 임의로 선택한다.

scheduler의 필터링, 스코어링 동작을 설정하는 데 지원하는 두 가지 방법이 있다:

1. scheduling policy을 사용하면 필터링을 위한 단정(Predicates) 및 스코어링을 위한 우선순위(Priorities) 를 구성할 수 있다.
2. scheduling profile을 사용하면 QueueSort, Filter, Score, Bind, Reserve, Permit 등의 다른 스케줄링 단계를 구현하는 플러그인을 설정할 수 있다. 다른 프로파일을 실행하도록 kube-scheduler를 설정할 수도 있다.

