k8s에서 스케줄링은 kubelet이 po를 실행할 수 있는 적합한 no를 찾는 것을 말한다.

## Scheduling overview
scheduler는 no가 할당되지 않은 새로 생성된 po를 감시한다. scheduler는 발견한 모든 po에 대해 해당 po가 실행될 가장 적합한 no를 찾는 책임을 갖는다. scheduler는 아래에 설명된 스케줄링 원칙을 고려해 배치 결정을 내린다.

특정 no에 po가 배치되는 이유를 이해하고자 하거나 사용자 정의 scheduler를 구현할 계획이 있다면 이 페이지를 통해 스케줄링에 대해 학습할 수 있다.

## kube-scheduler
[kube-scheduler](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-scheduler/)는 k8s의 기본 scheduler이며 control plane의 구성 요소로 실행된다. kube-scheduler는 필요에 따라 직접 사용자 정의 스케줄링 구성 요소를 개발해 사용할 수 있도록 설계됐다.

kube-scheduler는 scheduling frameowkr라는 plugin 아키텍처를 갖는다. kube-scheduler는 기본적으로 여러 plugin을 제공하며 추가적으로 사용자는 plugin을 개발하고 직접 kube-scheduler를 컴파일해서 사용할 수 있다. scheduling framework는 몇 가지 extension point을 정의한다. 스케줄러 plugin은 하나 이상의 extension point에서 호출되도록 등록한다. 이러한 plugin 중 일부는 스케줄링 결정을 변경할 수 있고 일부는 정보 제공에 불과하다. 하나의 po를 스케줄링하기 위한 프로세스는 2개의 phase(scheduling cycle, binding cycle)로 구성된다. scheduling cycle은 po를 위한 no를 선택하고 binding cycle은 해당 결정을 cluster에 적용한다. 이 두 cycle을 "scheduling context"라고 부른다.

scheduling cyle은 내부적으로 크게 filtering, scoring 단계로 수행된다. kube-scheduler는 새로 생성되거나 아직 스케줄링 되지 않은(unscheduled) po를 실행할 최적의 no를 선택한다. po 또는 container가 요구 사항을 가질 수 있기 때문에 scheduler는 po의 특정 스케줄링 요구 사항을 충족하지 않는 no를 걸러낸다(filtering). 물론 po를 생성할 때 no를 지정할 수도 있지만 이는 일반적이지 않으며 특수한 경우에만 사용한다.

위처럼 cluster에서 po의 스케줄링 요구 사항을 충족(filtering 단계)하는 no를 feasible no라고 한다. feasible no가 없는 경우 scheduler가 po를 배치할 수 있을 때까지 po는 unscheduled 상태로 유지된다.

scheduler는 po에 대한 feasible no를 모두 찾은 다음, 각 feasible no에 대해 일련의 함수를 실행해 점수를 계산하고, 가장 높은 점수를 받은 feasible no를 선택해 po를 할당한다. 동일한 점수를 가진 no가 여러 개인 경우 kube-scheduler는 이들 중 하나를 무작위로 선택한다. 그리고 scheduler는 이러한 결정을 `binding`이라는 프로세스에서 kube-apiserver에 통지한다. k8s 1.7 이전에는 k8s binding 내장 리소스를 사용했지만 1.7에서 deprecated 됐으며 대신 po의 binding subresource를 사용한다. binding subresource를 통해 po와 no다 연결(binding)되고, 이로 인해 po의 `.spec.nodeName` 필드에 해당 no의 이름이 설정된다.

스케줄링 결정에 고려해야 할 요소에는 리소스 요구 사항, 하드웨어/소프트웨어/policy constrains, affinity와 anti-affinity, data locality, workload 간 간섭 등이 포함된다.

### Node selection in kube-scheduler
kube-scheduler는 po를 no에 할당하는 데 2단계 작업을 수행한다.
1. Filtering
2. Scoring

filtering 단계에서는 po를 예약할 수 있는 no 집합을 찾는다. 예를 들어 `PodFitsResources` 필터는 후보 no가 po의 resource request을 충족할 수 있는 충분한 available resource를 가지고 있는지 확인한다. 이 단계 이후에는 적합한 no 목록이 구성된다. 만약 목록이 비어있다면 po는 아직 스케줄링이 불가한 상태다.

scoring 단계에서 scheduler는 가장 적합한 po의 배치를 위해 남은 no를 대상으로 순위를 매긴다. scheduler는 활성화된 scoring 규칙을 실행해 각 no에 점수를 할당한다.

결과적으로 kube-scheduler는 가장 높은 순위의 no에 po를 할당한다. 동일한 점수를 가진 no가 여러 개인 경우 kube-scheduler는 이들 중 하나를 무작위로 선택한다.

scheduler의 filtering, scoring 동작을 설정하는 두 가지 방법이 있다.
1. [Scheduling Policies](https://kubernetes.io/docs/reference/scheduling/policies): (k8s v1.23부터 미지원) filtering을 위한 조건(Predicates) 및 scoring를 위한 우선순위(Priorities)를 구성할 수 있다.
2. [Scheduling Profiles](https://kubernetes.io/docs/reference/scheduling/config/#profiles): 다양한 스케줄링 단계를 구현하는 플러그인을 구성할 수 있다. 여기에는 `QueueSort`, `Filter`, `Score`, `Bind`, `Reserve`, `Permit` 등이 포함된다. 또한 kube-scheduler를 다른 profile로 실행할 수도 있다.