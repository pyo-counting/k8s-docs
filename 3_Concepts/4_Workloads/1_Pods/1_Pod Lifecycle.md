po는 정의된 라이프 사이클을 따른다. pending phase를 시작으로 1개 이상의 주요 container가 OK 상태가 되면 running phase로 바뀌고 po 내 container의 실패로 인한 종료 여부에 따라 succeeded 또는 failed phase로 최종 바뀐다.

개별 애플리케이션 container와 마찬가지로 po 역시 임시 객체로 간주된다. po가 생성되면 고유 ID(UID)가 할당되고, no에 할당되어 종료(restartpolicy 정책에 따라 다름) 또는 삭제될 때까지 유지된다. 만약 no가 죽는다면, 해당 no에 있는 po를 삭제를 위해 marking된다.

## Pod lifetime
po가 실행 중인 동안 kubelet은 일부 실패 상황에 따라 container를 재시작한다. k8s는 po 내 각 container의 state(`.staus.containerStatuses`)를 추척하고 po가 다시 healthy 상태가 될 수 있도록 어떤 조치를 해야할지 결정한다.

k8s API에 po는 `.spec`과 실제 `.status`를 갖는다. po의 status는 `.status.conditions`을 포함한다. 사용자는 custom readiness information를 사용해 condition 정보를 추가할 수도 있다.

각 po는 한 번만 스케쥴링된다. po를 특정 no에 할당하는 것을 binding이라고 부르며 특정 no를 선택하는 과정을 scheduling이라고 부른다. po가 no에 스케쥴되면 해당 no에서 stop 또는 terminated까지 실행된다.

You can use Pod Scheduling Readiness to delay scheduling for a Pod until all its scheduling gates are removed. For example, you might want to define a set of Pods but only trigger scheduling once all the Pods have been created.

### Pods and fault recovery
po 내의 container 중 하나가 실패하면 k8s는 해당 container를 재시작하려고 시도한다. 상세 내용은 [How Pods handle problems with containers](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-restarts)을 참고한다.

그러나 po가 cluster가 복구할 수 없는 방식으로 실패할 수도 있으며, 이 경우 k8s는 더 이상 po를 복구하려고 시도하지 않는다. 대신, k8s는 po를 삭제하고 다른 구성 요소(컨트롤러)가 자동 복구를 처리하도록 한다.

po가 특정 no에 스케줄링된 후 해당 no에 장애가 발생하면, po는 비정상(unhealthy) 상태로 간주되고 k8s는 결국 해당 po를 삭제한다. 뿐만 아니라 po는 리소스 부족이나 no 유지보수로 인한 eviction 상황에서도 살아남지 못한다.

k8s는 controller라 불리는 더 높은 수준의 추상화를 사용해 상대적으로 일회용인(disposable) po 인스턴스를 관리하는 작업을 처리한다.

동일 UID를 갖는 po는 절대 다른 no에 "재스케줄링"되지 않는다. 대신, 새롭고 거의 동일한 po로 교체될 수 있다. 교체되는 새 po는 이전 po와 같은 이름(`.metadata.name`)을 가질 수는 있지만, 이전 po와는 다른 고유 식별자(`.metadata.uid`)를 갖는다.

k8s는 기존 po를 대체하는 새 po가 이전 po와 동일한 no에 스케줄링된다고 보장하지 않는다.

### Associated lifetimes

## Pod phase
po의 phase는 `.status.phase` 필드(PodStatus object)를 통해 정의된다.
``` yaml
status:
  phase: Running
  conditions:
    - type: PodReadyToStartContainers
      status: 'True'
      lastProbeTime: null
      lastTransitionTime: '2025-07-15T01:09:46Z'
    - type: Initialized
      status: 'True'
      lastProbeTime: null
      lastTransitionTime: '2025-07-15T01:09:45Z'
    - type: Ready
      status: 'True'
      lastProbeTime: null
      lastTransitionTime: '2025-07-15T01:10:19Z'
    - type: ContainersReady
      status: 'True'
      lastProbeTime: null
      lastTransitionTime: '2025-07-15T01:10:19Z'
    - type: PodScheduled
      status: 'True'
      lastProbeTime: null
      lastTransitionTime: '2025-07-15T01:09:45Z'
  hostIP: 172.16.52.137
  hostIPs:
    - ip: 172.16.52.137
  podIP: 172.16.52.209
  podIPs:
    - ip: 172.16.52.209
  startTime: '2025-07-15T01:09:45Z'
  containerStatuses:
    - name: alloy
      state:
        running:
          startedAt: '2025-07-15T01:10:07Z'
      lastState:
        terminated:
          exitCode: 1
          reason: Error
          startedAt: '2025-07-15T01:09:46Z'
          finishedAt: '2025-07-15T01:09:47Z'
          containerID: >-
            containerd://b5b2e252b67fa4d0fd9af3e1e099119e68029c6bd97b93143cdcc3494226969d
      ready: true
      restartCount: 2
      image: docker.io/grafana/alloy:v1.8.3
      imageID: >-
        docker.io/grafana/alloy@sha256:ab04df3936e4d71d31b6f55e0c58a7e749091f59635dd8c2bc731ba1b6c93701
      containerID: >-
        containerd://1bc1bc1df160258f7f1266d9a9fbc98a63b0ce53674be6093e6e8db92620d4af
      started: true
      volumeMounts:
        - name: config
          mountPath: /etc/alloy
        - name: varlog
          mountPath: /var/log
          readOnly: true
          recursiveReadOnly: Disabled
        - name: geolite2-city-db
          mountPath: /home/kurlypay
        - name: kube-api-access-cl9sg
          mountPath: /var/run/secrets/kubernetes.io/serviceaccount
          readOnly: true
          recursiveReadOnly: Disabled
  qosClass: Burstable
```

po의 phase는 라이프사이클 중 어떤 상태에 있는지에 대한 간단한 요약이다. phase는 container 또는 po의 관측 정보에 대한 포괄적인 상태를 표현하도록 의도되지는 않았다.

숫자와 pod phase 값의 의미는 엄격하다. 아래 문서화된 내용 이외에는 po와 po에 주어진 phase 값에 대해서 어떤 사항도 가정해서는 안된다.

phase에 가능한 값은 다음과 같다.
| 값        | 의미                                                                                                                                                                                          |
|-----------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Pending   | po가 k8s cluster에 승인됐지만, 1개 이상의 container가 실행할 준비가 되지않음. 이는 po가 스케쥴링되기를 기다리는 시간 뿐만 아니라 container 실행을 위한 image를 다운로드 하는 과정도 포함한다. |
| Running   | po가 특정 no에 할당되고 모든 container가 생성됐음을 의미한다. 적어도 1개의 container가 running 또는 starting 또는 restarting 중이다.                                                          |
| Succeeded | po의 모든 container가 success 상태로 종료됐으며 재시작되지 않는다.                                                                                                                            |
| Failed    | po의 모든 container가 종료됐으며 적어도 1개 이상의 container가 실패로 종료됐다. 즉, container가 시스템에 의해 종료됐거나 0이 아닌 status다.                                                   |
| Unknown   | 어떤 이유로 po의 state를 얻을 수 없다. 이 phase는 전형적으로 po가 실행 중인 no와 통신하는 데 오류가 발생할 경우다.                                                                            |

> **Note**:  
> kubectl 명령어 결과 Status 필드에 실제 po의 phase가 아닌 값을 보여주기도 한다.
> - CrashLoopBackOff: po가 계속 재시작 중일 때
> - Terminating: po가 gracefule shutdown period내(`.spec.terminationGracePeriodSeconds`) 있는 경우

k8s 1.27부터 kubelet은 static po와 finalizer가 없는 강제 삭제된 po를 제외하고 삭제될 po를 kube-apiserver에서 제거하기 전에 먼저 해당 po 내 container의 종료 상태에 따라 po의 phase를 업데이트(Failed 또는 Succeeded) 한다.

만약 no가 다운되거나 cluster와의 연결이 끊어지면, k8s는 해당 유실된 no 위에 있던 모든 po의 phase를 Failed로 설정하는 정책을 적용한다.

## Container states
po의 phase 뿐만 아니라 k8s는 po 내부의 각 container state(`.state.containerStatuses[*].state`)도 추적한다. container lifecycle hook을 사용하면 container lifecycle 중 특정 지점에서 실행할 event를 트리거할 수 있다.

container는 세 가지 state를 갖는다.
- **Waiting**: Waiting state의 container는 실행을 완료하는 데 필요한 작업(container image pull 작업, secret 데이터 적용)을 계속 실행하고 있는 중이다. kubectl describe 명령어를 사용해 Waiting state에 있는 이유를 나타내는 Reason 필드를 확인할 수 있다.
- **Running**: Running state는 container가 문제 없이 동작 중이다. postStart hook이 설정된 경우 완료된 상태다.
- **Terminated**: Terminated state의 container는 실행을 완료했거나 어떤 이유로 실패한 것이다. kubectl describe 명령어를 사용해 종료 이유, exit code, 시작/종료 시간을 확인할 수 있다. container에 preStop hook이 설정된 경우 container가 Termindate state에 돌입하기 전에 실행된다.

## How Pods handle problems with containers
k8s는 po `.spec.restartPolicy`을 사용하여 po 내 container의 장애를 관리한다. 이 정책은 오류나 다른 이유로 container가 종료될 때 k8s가 어떻게 반응할지를 결정하며, 다음과 같은 순서로 진행한다.
1. Initial crash: container에 문제가 발생하면, k8s는 po의 `.spec.restartPolicy`에 따라 즉시 재시작을 시도한다.
2. Repeated crashes: Initial crash 이후에도 container가 계속해서 실패하면, k8s는 후속 재시작에 대해 exponential back-off 지연을 적용한다. 이는 시스템에 과부하를 주는 무분별하고 반복적인 재시작 시도를 방지한다.
3. CrashLoopBackOff state: 이 상태는 현재 exponential back-off 메커니즘이 특정 container에 적용되고 있음을 나타낸다. 해당 container는 반복적으로 실패하고 재시작하는 '크래시 루프(crash loop)'에 빠져 있다.
4. Backoff reset: 만약 container가 특정 시간(10분) 동안 성공적으로 실행되면, k8s는 exponential back-off 지연 시간을 초기화하고, 이후에 발생하는 새로운 크래시는 첫 번째 크래시로 간주한다.

실제 환경에서 CrashLoopBackOff는, po 내의 container가 제대로 시작되지 못하고 계속해서 시작과 실패를 반복하는 루프에 빠졌을 때 kubectl 명령어로 po를 조회하거나 상세 정보를 볼 때 나타나는 상태 또는 이벤트다.

다시 말해, container가 크래시 루프에 진입하면 k8s는 exponential back-off 지연을 적용한다. 이 메커니즘은 결함이 있는 container가 지속적인 실패와 재시작 시도로 시스템에 영향을 미치는 것을 방지한다.

CrashLoopBackOff는 다음과 같은 문제들로 인해 발생할 수 있다.
- container를 종료시키는 애플리케이션 오류
- 잘못된 환경 변수나 설정 파일 누락과 같은 설정 오류
- container가 제대로 시작하기에 메모리나 CPU가 충분하지 않은 리소스 제약
- 애플리케이션이 예상 시간 내에 서비스를 시작하지 못해 발생하는 health check 실패
- liveness probe나 startup probe가 Failure 결과를 반환하는 경우

CrashLoopBackOff 문제의 근본 원인을 조사하기 위해 사용자는 다음을 수행할 수 있다.
- 로그 확인: kubectl logs <pod-name>를 사용하여 container의 로그를 확인한다. 이는 크래시를 유발하는 문제를 진단하는 가장 직접적인 방법인 경우가 많다.
- 이벤트 검사: kubectl describe pod <pod-name>를 사용하여 po의 이벤트를 확인한다. 이는 설정이나 리소스 문제에 대한 힌트를 제공할 수 있다.
- 설정 검토: 환경 변수, 마운트된 볼륨을 포함한 po의 구성이 올바른지, 필요한 모든 외부 리소스를 사용할 수 있는지 확인한다.
- 리소스 제한 확인: container에 충분한 CPU와 메모리가 할당되었는지 확인합니다. 때로는 po 정의에서 리소스를 늘리면 문제가 해결될 수 있다.
- 애플리케이션 디버깅: 애플리케이션 코드에 버그나 잘못된 구성이 있을 수 있다. 해당 container 이미지를 로컬이나 개발 환경에서 실행하면 애플리케이션 특정 문제를 진단하는 데 도움이 될 수 있다.

### Container restart policy
po의 spec에는 restartPolicy 필드가 있다. Always, OnFailure, Never 값을 가질 수 있으며 기본값은 Always다.

`.spec.restartPolicy`는 po 내 애플리케이션 container, init container에 적용된다. sidecar container는 po 레벨의 restartPolicy 필드를 무시한다. sidecar container는 init container 중 `.spec.initContainers[*].restartPolicy`(해당 필드 init container에만 사용 가능하며 값은 Always만 사용 가능) 가 Always인 container다. init container가 성공 종료하지 못한다면 po의 `.spec.restartPolicy`가 OnFailure, Always일 경우에만 재실행한다.
- Always: Automatically restarts the container after any termination.
- OnFailure: Only restarts the container if it exits with an error (non-zero exit status).
- Never: Does not automatically restart the terminated container.

container가 종료된 후 exponential back-off 지연(10s, 20s, 40s, ...최대 5분)을 갖고 재시작된다. container가 실행된 후 10분 간 문제가 없으면 kubelet은 container에 대한 restart backoff timer를 초기화한다.

### Reduced container restart delay

### Configurable container restart delay

## Pod conditions
po는 특정 상태 조건을 충족했는지 여부를 나타내는 `.status.conditions[*]` 필드(PodConditions 배열)를 가진다. 아래는 condition 목록이다.
- PodScheduled: po가 no에 스케줄됐다.
- PodReadyToStartContainers: po sandbox(pause container)가 성공적으로 생성 및 네트워크 설정이 완료됐다.
- ContainersReady: po의 모든 container가 준비되었다.
- Initialized: 모든 init container가 성공적으로 완료(completed)됐다.
- Ready: po는 요청을 처리할 수 있으며 매칭되는 모든 svc의 로드 밸런싱 풀에 추가되어야 한다.
- DisruptionTarget: disruption(preemption, eviction, gc)으로 po가 종료되려고 한다.
- PodResizePending: po의 resize가 요청됐지만 적용될 수 없다.
- PodResizeInProgress: po의 resize가 진행 중이다.

| 필드 명            | 설명                                                              |
|--------------------|-------------------------------------------------------------------|
| type               | po condition 이름                                                 |
| status             | condition 적용 가능 여부(True, False, Unknown) 값을 가질 수 있다. |
| lastProbeTime      | po condition의 마지막 probe 시간                                  |
| lastTransitionTime | po의 상태의 마지막 변경된 시간                                    |
| reason             | condition 마지막 변경에 대한 machine-readable 이유                |
| message            | condition 마지막 변경에 대한 human-readable 이유                  |

### Pod readiness
애플리케이션을 위한 pod readiness를 PodStatus에 추가할 수 있다: 사용자는 kubelet이 po의 readiness를 평가하기 위한 추가 조건을 po의 `.spec.readinessGates[*].`에 명시할 수 있다. readinessGates는 po의 준비 상태를 결정할 때, container 자체의 준비 상태 외에 추가적인 외부 조건들까지 고려하게 해주는 기능이다.

readiness gate는 po의 `.status.condition` 필드의 상태에 따라 결정된다. 만약 `.status.condtion` 필드에서 해당 필드를 확인하지 못하면 기본 값 `False`로 인지한다. 아래는 예시다.
``` yaml
kind: Pod
...
spec:
  readinessGates:
    - conditionType: "www.example.com/feature-1"
status:
  conditions:
    - type: Ready                              # a built in PodCondition
      status: "False"
      lastProbeTime: null
      lastTransitionTime: 2018-01-01T00:00:00Z
    - type: "www.example.com/feature-1"        # an extra PodCondition
      status: "False"
      lastProbeTime: null
      lastTransitionTime: 2018-01-01T00:00:00Z
  containerStatuses:
    - containerID: docker://abcd...
      ready: true
...
```

추가한 po condition의 이름은 [label key format](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/#syntax-and-character-set)을 따라야한다.

### Status for Pod readiness
`kubectl patch` 명령어는 `.status` 필드에 대한 patch 작업을 지원하지 않는다. `.status.conditions`를 설정하기 위해 application 또는 operator가 `PATCH` action을 수행해야 한다. 사용자는 po의 readiness를 위한 custom po condition을 설정하기 위해 [Kubernetes client library](https://kubernetes.io/docs/reference/using-api/client-libraries/)를 사용할 수 있다.

custom condition을 사용하는 po는 아래 조건을 모두 만족하는 경우에 대해서만 ready로 평가된다.
- po 내 모든 container가 Ready 상태
- `.spec.readinessGates[*]`에 명시된 모든 condition이 `True`일 때

po의 모든 container는 Ready 상태가 되었지만(readinessProbe 통과), readinessGates에 지정된 커스텀 조건 중 하나라도 False이거나 아직 설정되지 않았다면 kubelet은 po의 전체 상태는 "Not Ready"로 두지만, Pod의 conditions 필드에 ContainersReady라는 특별한 상태를 True로 설정한다. 이는 "po 내부의 애플리케이션들은 준비되었지만, 외부 의존성과 관련된 준비 조건이 아직 충족되지 않았다"는 것을 명확히 알려주는 유용한 신호다.

### Pod network readiness

## Container probes
probe(`.spec.containers[*].lifecycle` 필드)는 kubelet이 container에 대해 주기적으로 진단을 수행한다. 이를 위해 kubelet은 container 내에서 코드를 실행하거나 container를 대상으로 네트워크 요청을 전송한다.

### Check machanisms
4가지 메커니즘을 통해 probe를 정의할 수 있다.
- exec:
- grpc:
- httpGet:
- tcpSocket:

> **Caution**:
> 다른 메커니즘과 달리, exec probe의 구현에는 실행될 때마다 여러 프로세스를 생성하고 복제(fork)하는 과정이 포함된다. 결과적으로, po 밀도가 높거나 initialDelaySeconds, periodSeconds의 간격이 짧은 cluster에서 exec 방식의 probe를 설정하면 no의 CPU 사용량에 오버헤드를 유발할 수 있다. 이러한 시나리오에서는 오버헤드를 방지하기 위해 대체 probe 메커니즘(HTTP 또는 TCP 프로브) 사용을 고려해야 한다.

### Probe outcome
3가지 probe 결과가 있다.
- Success:
- Failure:
- Unknown:

### Types of probe
실행 중인 container에 대해 kubelet은 아래 유형의 probe를 수행할 수 있다:

- livenessProbe(`.spec.containers[*].lifecycle.livenessProbe`): container가 구동 중인지 확인한다. liveness probe가 실패하면 kubelet은 container를 kill하고 restart policy에 따라 재시작한다. liveness probe가 없다면 기본 state는 Success다.
- readinessProbe(`.spec.containers[*].lifecycle.readinessProbe`): container가 요청에 대한 응답을 할 준비가 됐는지 확인한다. readiness probe가 실패하면 ep controller는 해당 po와 관련된 svc의 ep에서 해당 po의 IP를 모두 삭제한다. initial delay 전까지 기본 state는 Failure다. readiness probe가 없다면 기본 state는 Success다.
- startupProbe(`.spec.containers[*].lifecycle.startupProbe`): container 내 애플리케이션이 정상 실행됐는지 확인한다. startup probe가 성공하기 전까지 다른 모든 probe는 비활성화된다. startup probe가 실패하면 kubelet은 container를 종료하고 restart policy에 따라 재시작한다. startup probe가 없다면 기본 state는 Success다.

#### When should you use a liveness probe?
만약 container 내 프로세스가 어떠한 이슈에 직면하거나 unhealthy 상태가 되는 등 프로세스 자체의 문제로 중단될 수 있더라도 liveness probe가 반드시 필요한 것은 아니다. kubelet이 po의 restartPolicy에 따라서 올바른 대처를 자동적으로 수행할 것이다.

probe가 실패한 후 container가 종료되거나 재시작되길 원한다면 liveness probe를 설정하고, restartPolicy를 Always 또는 OnFailure로 지정한다

#### When should you use a readiness probe?
probe가 성공했을 때만 po에 트래픽을 보내고 싶다면 readiness probe를 사용하면된다. 이 경우 readiness probe는 liveness probe와 동일할 수 있지만, spec에 readiness probe가 존재한다는 것은 po가 처음에는 트래픽을 받지 않고 시작하며 probe가 성공하기 시작한 이후에야 트래픽을 받기 시작한다는 것을 의미한다.

container가 유지보수를 위해 스스로 서비스에서 제외되도록 하고 싶다면, liveness probe와는 다른 readiness 전용 엔드포인트를 확인하는 readiness probe를 지정할 수 있다.

앱이 백엔드 서비스에 강하게 의존한다면 liveness와 readiness probe를 모두 구현할 수 있다. liveness probe는 앱 자체가 healthy할 때 통과하지만, readiness probe는 추가적으로 필요한 각 백엔드 서비스가 사용 가능한지까지 확인한다. 이를 통해 오류 메시지만 응답할 수 있는 po로 트래픽이 전달되는 것을 방지할 수 있다.

container가 시작 중에 대용량 데이터, 설정 파일, 또는 마이그레이션 작업을 해야 한다면, startup probe를 사용할 수 있다. 그러나 실패한 앱과 아직 시작 데이터를 처리 중인 앱의 차이를 감지하고 싶다면 readiness probe를 선호할 수도 있다.

> **Note**:  
> po가 삭제될 때 요청을 안전하게 처리하고 종료(draining)하고 싶다고 해서 반드시 readiness probe가 필요한 것은 아니다. po가 삭제되면, 해당하는 es의 엔드포인트 conditions가 업데이트된다: 엔드포인트의 ready 조건이 false로 설정되므로, 로드 밸런서는 더 이상 해당 po를 일반 트래픽에 사용하지 않는다. kubelet이 po 삭제를 처리하는 방법에 대한 자세한 내용은 [Pod termination](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination) 문서를 참고한다.

#### When should you use a startup probe?
startup probe는 서비스를 시작하는 데 오랜 시간이 걸리는 container가 있는 po에 유용하다. 긴 liveness 간격을 설정하는 대신 container가 시작될 때 probe를 위한 별도의 값을 설정해 liveness probe보다 긴 시간을 허용할 수 있다.

container가 보통 initialDelaySeconds + failureThreshold × periodSeconds 이후에 기동된다면, startup probe가 liveness probe와 같은 엔드포인트를 확인하도록 지정해야 한다. periodSeconds의 기본값은 10s 이다. 이 때 container가 liveness probe의 기본값 변경 없이 기동되도록 하려면, failureThreshold 를 충분히 높게 설정해주어야 한다. 그래야 데드락(deadlocks)을 방지하는데 도움이 된다.

## Termination of Pods
po는 클러스터 내에서 no에서 실행되는 프로세스를 나타내기 때문에 KILL signal로 종료해 gracefully shutdown하지 못하도록하는 것이 아니라 gracefully shutdown하는 것이 중요하다.

이 설계의 목적은 삭제를 요청할 수 있게하고, 프로세스의 종료시점을 알게하고, 삭제가 완료됐음을 확인하게하는 것이다. po 삭제를 요청하면 cluster는 po가 강제 종료되기 전에 의도한 grace period를 기록하고 추적한다. 강제 종료 추적이 설정된 상태에서 kubelet은 graceful shutdown을 시도한다.

일반적으로 kubelet은 container runtime을 통해 각 container의 메인 프로세스에 grace period timeout이 있는 TERM(또는 SIGTERM) signal을 먼저 전송해 po의 container를 먼저 종료한다. container 종료 요청은 container runtime에 의해 비동기적으로 처리된다. 이러한 요청에 대한 처리 순서는 보장되지 않는다. 대부분의 container runtime은 container image에 정의된 STOPSIGNAL 명령어에 설정된 signal을 TERM signal 대신 사용한다. grace period가 만료되면, KILL signal이 남아있는 모든 프로세스에 전송되고 po는 kube-apiserver에서 삭제된다. 프로세스가 종료될 때까지 기다리는 동안 kubelet 또는 container runtime 관리 서비스가 다시 시작되면 cluster는 전체 grace period을 포함하여 처음부터 다시 시도한다.

### Stop Signals
container를 종료시키는 데 사용되는 stop signal은 container image 내 STOPSIGNAL 명령어를 통해 정의할 수 있다. 만약 image에 stop signal이 정의되어 있지 않다면 container를 종료시키기 위해 container runtime의 기본 신호(containerd와 CRI-O 모두 SIGTERM)가 사용된다.

### Defining custom stop signals

### Pod Termination Flow
아래는 po 종료에 대한 예시다.
1. kubectl을 사용해 po를 삭제한다(grace period(`.spec.terminationGracePeriodSeconds`)의 기본 값은 30s).
2. kube-apiserver 내에서 po는 grace period와 함께 "dead"로 간주되는 시간으로 업데이트된다(grace period의 countdown이 시작된다). kubectl describe로 확인 시 po는 "Terminating"으로 표시된다. po가 실행되는 no의 kubelet은 해당 po가 terminating으로 표시된 것을 확인하고 po의 종료 프로세스를 시작한다.
    1. container가 preStop hook을 설정한 경우, kubelet은 container 내부에서 hook을 실행한다. grace period가 만료된 후 preStop hook이 계속 실행 중이라면, kubelet은 2초의 작은 일회성 grace period 연장을 요청한다.
        > **Note**:  
        > preStop ho이 필요한 경우 `.spec.terminatiook을 완료하는 데 기본 grace period가 허용하는 것보다 오랜 시간nGracePeriodSeconds`을 수정한다.
    2. preStop hook 실행이 완료된 후, kubelet은 container runtime을 트리거해 각 container 내부 1번 프로세스에 TERM signal을 전송한다.
        > **Note**:  
        > sidecar container를 사용하는 경우 정해진 순서가 있다. 그렇지 않으면 po의 container는 서로 다른 시간에 임의 순서로 TERM signal을 수신한다. 종료 순서가 중요한 경우 preStop hook을 사용해 동기화하면 된다.
3. kubelet이 graceful shutdown을 실행하는 것과 동시에 control plane은 svc의 ep에서 해당 po를 제거한다. rs과 같은 workload resoucre는 더 이상 해당 po를 유효하다고 판단하지 않는다.   
Pods that shut down slowly should not continue to serve regular traffic and should start terminating and finish processing open connections. Some applications need to go beyond finishing open connections and need more graceful termination, for example, session draining and completion.   
Any endpoints that represent the terminating Pods are not immediately removed from EndpointSlices, and a status indicating terminating state is exposed from the EndpointSlice API (and the legacy Endpoints API). Terminating endpoints always have their ready status as false (for backward compatibility with versions before 1.26), so load balancers will not use it for regular traffic.   
If traffic draining on terminating Pod is needed, the actual readiness can be checked as a condition serving. You can find more details on how to implement connections draining in the tutorial Pods And Endpoints Termination Flow
4. kubelet은 po가 완전히 종료되는 것을 보장하기 위해 다음과 같은 동작을 수행한다.
  1. grace period가 완료됐음에도 실행 중인 container가 있는 경우 forcible shutdown을 트리거한다. container runtime은 container내 실행 중인 모든 프로세스에 SIGKILL signal을 전송한다. 또한 kubelet은 pause container가 있는 container runtime에 대해 해당 container도 정리한다.
  2. po를 terminal phase(`Failed` 또는 `Succeeded`)로 변경한다.
  3. kubelet은 grace period를 0로 변경함으로써 po를 force deletion한다.
  4. kube-apiserver는 po object를 삭제한다.

### Forced Pod termination
> **Caution**:  
> Forced deletions can be potentially disruptive for some workloads and their Pods.
기본 삭제에 대한 grace period는 30초다. kubectl delete 명령어의 --grace-period flag를 사용해 변경할 수 있다.

grace period를 0으로 설정하면 kube-apiserver에서 po가 즉시 삭제된다. 만약 no에 po가 실행 중이라면 kubelet은 즉시 종료 프로세스를 시작한다.

kubectl을 사용하는 경우 --force, --grace-period=0을 사용해 force deletion을 수행할 수 있다.

force deletion이 수행되면, kube-apiserver는 no에서 po가 종료되었다는 kubelet의 확인을 기다리지 않는다. kube-apiserver에서 즉시 po를 제거하므로 동일한 이름으로 새로운 po를 생성할 수 있다. 즉시 종료되도록 설정된 po는 no에서 강제 종료되기 전에 짧은 grace period가 제공된다.

> **Caution**:  
> immediate deletion은 실행 중인 po를 즉시 종료 및 삭제하지만 실제로 종료가 됐는지 여부는 확인하지 않는다. 그렇기 때문에 해당 po가 no에 계속 남아있을 수도 있게 된다.

If you need to force-delete Pods that are part of a StatefulSet, refer to the task documentation for [deleting Pods from a StatefulSet](https://kubernetes.io/docs/tasks/run-application/force-delete-stateful-set-pod/).

### Pod shutdown and sidecar containers
po에 하나 이상의 sidecar container(Always 재시작 정책을 가진 init container)가 포함된 경우, kubelet은 마지막 메인 container가 완전히 종료될 때까지 sidecar container에 TERM 신호를 보내는 것을 지연한다. sidecar container는 po spec에 정의된 역순으로 종료된다. 이는 sidecar container가 더 이상 필요하지 않을 때까지 po 내의 다른 container에 계속 서비스를 제공하도록 보장하기 위함이다.

이는 메인 container의 느린 종료가 sidecar container의 종료 또한 지연시킨다는 것을 의미한다. 만약 종료 프로세스가 완료되기 전에 grace period가 만료되면 po는 강제 종료(forced deletion)될 수 있다. 이 경우 po에 남아있는 모든 container는 짧은 grace period과 함께 동시에 종료된다.

유사하게 po가 종료 grace period을 초과하는 preStop hook을 가지고 있다면 긴급 종료가 발생할 수 있다. 만약 종료 순서를 제어하기 위해 preStop hook을 사용해왔다면, 이제는 kubelet이 sidecar를 사용해 종료를 자동으로 관리하도록 할 수 있다.

### Garbage collection of failed Pods
실패한 po의 경우 사용자 또는 controller가 명시적으로 삭제할 때까지 kube-apiserver에 존재한다.

kube-controller-manager 내 pod garbage collector(PodGC)는 terminated po(phase가 `Succeeded`, `Failed`)의 수가 --terminated-pod-gc-threshold(기본 값 12500)을 초과할 때 정리한다.

뿐만 아니라 아래 조건을 만족하는 po도 정리한다.
- orphan po: 더 이상 존재하지 않은 no에 binding된 경우
- 스케줄되지 않은 종료 중인 po
- `node.kubernetes.io/out-of-service` taint가 있는 non-ready no에 있는 종료 중인 po

PodGC는 po를 정리하는 과정에서, 아직 종료 단계(non-terminal phase)가 아닌 po를 Failed로 표시하기도 한다. 또한, orphan po를 정리할 때는 'po 중단 조건(Pod disruption condition)'을 추가한다. 자세한 내용은 [Pod disruption conditions](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/#pod-disruption-conditions)을 참고한다.