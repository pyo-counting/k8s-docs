po는 정의된 라이프 사이클을 따른다. pending phase를 시작으로 1개 이상의 주요 container가 OK 상태가 되면 running phase로 바뀌고 po 내 container의 실패로 인한 종료 여부에 따라 succeeded 또는 failed phase로 최종 바뀐다.

po가 실행 중에 kubelet은 일부 실패 상황에 따라 container를 재시작한다. k8s는 po 내 각 container의 state(`.staus.containerStatuses`)를 추척하고 po가 다시 healthy 상태가 될 수 있도록 어떤 조치를 해야할지 결정한다.

k8s API에 po는 `.spec`과 실제 `.status`를 갖는다. po의 status는 `.status.conditions`을 포함한다. condition data에 사용자가 custom readiness information를 추가할 수도 있다.

각 po는 한 번만 스케쥴링된다. po가 no에 스케쥴되면 해당 no에서 stop 또는 terminated까지 실행된다.

``` yaml
status:
  conditions:
  - lastProbeTime: null
    lastTransitionTime: "2022-02-17T21:51:01Z"
    status: "True"
    type: Initialized
  - lastProbeTime: null
    lastTransitionTime: "2022-02-17T21:51:06Z"
    status: "True"
    type: Ready
  - lastProbeTime: null
    lastTransitionTime: "2022-02-17T21:51:06Z"
    status: "True"
    type: ContainersReady
  - lastProbeTime: null
    lastTransitionTime: "2022-02-17T21:51:01Z"
    status: "True"
    type: PodScheduled
  containerStatuses:
  - containerID: containerd://5403af59a2b46ee5a23fb0ae4b1e077f7ca5c5fb7af16e1ab21c00e0e616462a
    image: docker.io/library/nginx:latest
    imageID: docker.io/library/nginx@sha256:2834dc507516af02784808c5f48b7cbe38b8ed5d0f4837f16e78d00deb7e7767
    lastState: {}
    name: nginx
    ready: true
    restartCount: 0
    started: true
    state:
      running:
        startedAt: "2022-02-17T21:51:05Z"
  hostIP: 192.168.0.113
  phase: Running
  podIP: 10.88.0.3
  podIPs:
  - ip: 10.88.0.3
  - ip: 2001:db8::1
  qosClass: Guaranteed
  startTime: "2022-02-17T21:51:01Z"
```

## Pod lifetime
개별 애플리케이션 container와 마찬가지로 po 역시 임시 객체로 간주된다. po가 생성되면 고유 ID(UID)가 할당되고, no에 할당되어 종료 또는 삭제될 때까지 유지된다. 만약 no가 죽는다면, po는 timeout 시간이 지난 후 삭제에 대한 작업이 스케쥴링된다.

volume은 po와 동일한 라이프타임을 갖는다.

## Pod phase
po의 phase는 .status.phase 필드(PodStatus object)를 통해 정의된다.

po의 phase는 라이프사이클 중 어떤 상태에 있는지에 대한 간단한 요약이다. phase는 container 또는 po의 관측 정보에 대한 포괄적인 상태를 표현하도록 의도되지는 않았다.

숫자와 pod phase 값의 의미는 엄격하다. 아래 문서화된 내용 이외에는 po와 po에 주어진 phase 값에 대해서 어떤 사항도 가정해서는 안된다.

phase에 가능한 값은 다음과 같다.

|값|의미|
|---|---|
|Pending|po가 k8s cluster에 승인됐지만, 1개 이상의 container가 실행할 준비가 되지않음. 이는 po가 스케쥴링되기를 기다리는 시간 뿐만 아니라 container 실행을 위한 image를 다운로드 하는 과정도 포함한다.|
|Running|po가 특정 no에 할당되고 모든 container가 생성됐음을 의미한다. 적어도 1개의 container가 running 또는 starting 또는 restarting 중이다.|
|Succeeded|po의 모든 container가 success 상태로 종료됐으며 재시작되지 않는다.|
|Failed|po의 모든 container가 종료됐으며 적어도 1개 이상의 container가 실패로 종료됐다. 즉, container가 시스템에 의해 종료됐거나 0이 아닌 status다.|
|Unknown|어떤 이유로 Pod의 state를 얻을 수 없다. 이 phase는 전형적으로 po가 실행 중인 no와 통신하는 데 오류가 발생할 경우다.|

**Note:** po가 삭제될 때 몇몇 kubectl 명령어는 Terminating 상태를 노출한다. Terminating은 po의 phase가 아니다. 이 때 po는 grafecully 종료되도록 기간이 부여되며, 기본 값은 30초다. 강제로 po를 종료하기 위해 --force flag를 사용할 수 있다.

no가 죽거나 cluster에서 연결이 종료되면, k8s는 손실된 no의 모든 po phase를 Failed로 설정하는 정책을 적용한다.

## Container states
po의 phase 뿐만 아니라 k8s는 po 내부의 각 container state(`.state.containerStatuses[*].state`)도 추적한다. container lifecycle hook을 사용하면 container lifecycle 중 특정 지점에서 실행할 event를 트리거할 수 있다.

container는 세 가지 state를 갖는다.

- **Waiting**: Waiting state의 container는 실행을 완료하는 데 필요한 작업(container image pull 작업, secret 데이터 적용)을 계속 실행하고 있는 중이다. kubectl describe 명령어를 사용해 Waiting state에 있는 이유를 나타내는 Reason 필드를 확인할 수 있다.
- **Running**: Running state는 container가 문제 없이 동작 중이다. postStart hook이 설정된 경우 완료된 상태다.
- **Terminated**: Terminated state의 container는 실행을 완료했거나 어떤 이유로 실패한 것이다. kubectl describe 명령어를 사용해 종료 이유, exit code, 시작/종료 시간을 확인할 수 있다. container에 preStop hook이 설정된 경우 container가 Termindate state에 돌입하기 전에 실행된다.

## Container restart policy
po의 spec에는 restartPolicy 필드가 있다. Always, OnFailure, Never 값을 가질 수 있으며 기본값은 Always다.

restartPolicy는 po 내 모든 container에 적용된다. container가 종료된 후 exponential back-off 지연(10s, 20s, 40s, ...최대 5분)을 갖고 재시작된다. container가 실행된 후 10분 간 문제가 없으면 kubelet은 container에 대한 restart backoff timer를 초기화한다.

### Pod conditions
po는 PodStatus 오브젝트를 가지며 po의 통과 여부를 나타내는 .status.conditions[*] 필드(PodConditions 배열)를 가진다.

- PodScheduled: po가 no에 스케줄되었다.
- ContainersReady: po의 모든 container가 준비되었다.
- Initialized: 모든 init container가 성공적으로 완료(completed)되었다.
- Ready: po는 요청을 처리할 수 있으며 매칭되는 모든 Service의 로드 밸런싱 풀에 추가되어야 한다.

|필드 명|설명|
|---|---|
|type|po condition 이름|
|status|condition 적용 가능 여부(True, False, Unknown 값을 가질 수 있다.|
|lastProbeTime|po condition의 마지막 probe 시간|
|lastTransitionTime|po의 상태의 마지막 변경된  시간|
|reason|condition 마지막 변경에 대한 machine-readable 이유|
|message|condition 마지막 변경에 대한 human-readable 이유|

### Pod readiness
애플리케이션을 위한 Pod readiness를 PodStatus에 추가할 수 있다. 이를 사용하기 위해 kubelet이 po readiness를 평가하기 위한 추가 condition들을 po의 .spec.readinessGates 필드에 추가 할 수 있다.

Readiness gates are determined by the current state of .status.condition fields for the Pod. If Kubernetes cannot find such a condition in the status.conditions field of a Pod, the status of the condition is defaulted to "False".

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

### Status for Pod readiness
The kubectl patch command does not support patching object status. To set these status.conditions for the pod, applications and operators should use the PATCH action. You can use a Kubernetes client library to write code that sets custom Pod conditions for Pod readiness.

For a Pod that uses custom conditions, that Pod is evaluated to be ready only when both the following statements apply:

- All containers in the Pod are ready.
- All conditions specified in readinessGates are True.

When a Pod's containers are Ready but at least one custom condition is missing or False, the kubelet sets the Pod's condition to ContainersReady.

### Pod network readiness

## Container probes
probe는 kubelet이 container에 대해 주기적으로 진단을 수행한다. 이를 위해 kubelet은 container 내에서 코드를 실행하거나 container를 대상으로 네트워크 요청을 전송한다:

### Check machanisms
4가지 메커니즘을 통해 probe를 정의할 수 있다:

- `exec`: 
- `grpc`:
- `httpGet`:
- `tcpSocket`:

### Probe outcome
3가지 probe 결과가 있다:

- `Success`:
- `Failure`:
- `Unknown`:

### Types of probe
실행 중인 container에 대해 kubelet은 아래 유형의 probe를 수행할 수 있다:

- `livenessProbe`: container가 구동 중인지 확인한다.. liveness probe가 실패하면 kubelet은 container를 kill하고 restart policy에 따라 재시작한다. liveness probe가 없다면 기본 state는 Success다.
- `readinessProbe`: container가 요청에 대한 응답을 할 준비가 됐는지 확인한다. readiness probe가 실패하면 ep controller는 해당 po와 관련된 svc의 ep에서 해당 po의 IP를 모두 삭제한다. initial delay 전까지 기본 state는 Failure다. readiness probe가 없다면 기본 state는 Success다.
- `startupProbe`: container 내 애플리케이션이 정상 실행됐는지 확인한다. startup probe가 성공하기 전까지 다른 모든 probe는 비활성화된다. startup probe가 실패하면 kubelet은 container를 종료하고 restart policy에 따라 재시작한다. startup probe가 없다면 기본 state는 Success다.    

#### When should you use a liveness probe?
만약 container 내 프로세스가 어떠한 이슈에 직면하거나 unhealthy 상태가 되는 등 프로세스 자체의 문제로 중단될 수 있더라도 liveness probe가 반드시 필요한 것은 아니다; kubelet이 po의 restartPolicy에 따라서 올바른 대처를 자동적으로 수행할 것이다.

probe가 실패한 후 container가 종료되거나 재시작되길 원한다면 liveness probe를 설정하고, restartPolicy를 Always 또는 OnFailure로 지정한다

#### When should you use a readiness probe?

#### When should you use a startup probe?
startup probe는 서비스를 시작하는 데 오랜 시간이 걸리는 container가 있는 po에 유용하다. 긴 liveness 간격을 설정하는 대신 container가 시작될 때 probe를 위한 별도의 값을 설정해 liveness probe보다 긴 시간을 허용할 수 있다.

container가 보통 initialDelaySeconds + failureThreshold × periodSeconds 이후에 기동된다면, startup probe가 liveness probe와 같은 엔드포인트를 확인하도록 지정해야 한다. periodSeconds의 기본값은 10s 이다. 이 때 container가 liveness probe의 기본값 변경 없이 기동되도록 하려면, failureThreshold 를 충분히 높게 설정해주어야 한다. 그래야 데드락(deadlocks)을 방지하는데 도움이 된다.

## Termindateion of Pods
디자인 목표는 삭제를 요청하고 프로세스가 종료되는 시점을 알 수 있도록 하는 동시에 동시에 삭제가 결국 완료될 수 있도록 하는 것이다. po 삭제를 요청하면 클러스터는 po가 강제 종료되기 전에 의도한 grace period를 기록하고 추적한다. 강제 종료 추적이 설정된 상태에서 kubelet은 graceful shutdown을 시도한다.

일반적으로 container runtime은 각 container의 메인 프로세스에 TERM signal를 전송한다. 대부분의 container runtime은 container image에 정의된 STOPSIGNAL에 설정된 signal을 TERM signal 대신 사용한다. grace period가 만료되면, KILL signal이 남아있는 모든 프로세스에 전송되고 po는 API server에서 삭제된다. 프로세스가 종료될 때까지 기다리는 동안 kubelet 또는 container runtime 관리 서비스가 다시 시작되면 클러스터는 전체 grace period을 포함하여 처음부터 다시 시도한다.

아래는 플로우 예시다:

1. kubectl을 사용해 po를 삭제한다(grace period 기본 값 30초 사용).
2. API server 내에서 po는 grace period와 함께 "dead"로 간주되는 시간으로 업데이트된다. kubectl describe로 확인 시 po는 "Terminating"으로 표시된다. po가 실행되는 no의 kubelet은 해당 po가 terminating으로 표시된 것을 확인하고 po의 종료 프로세스를 시작한다.
    1. container가 preStop hook을 설정한 경우, kubelet은 container 내부에서 hook을 실행한다. grace period가 만료된 후 preStop hook이 계속 실행 중이라면, kubelet은 2초의 작은 일회성 grace period 연장을 요청한다.
        **Note**: preStop hook을 완료하는 데 기본 grace period가 허용하는 것보다 오랜 시간이 필요한 경우 terminationGracePeriodSeconds을 수정한다. 
    2. kubelet은 container runtime을 트리거해 각 container 내부 1번 프로세스에 TERM signal을 전송한다.
        **Note**: po의 container는 서로 다른 시간에 임의 순서로 TERM signal을 수신한다. 종료 순서가 중요한 경우 preStop hook을 사용해 동기화하면 된다.
3. kubelet이 graceful shutdown을 실행하는 것과 동시에 control plane은 svc의 ep에서 해당 po를 제거한다. rs과 같은 workload resoucre는 더 이상 해당 po를 유효하다고 판단하지 않는다. 
4. grace period가 만료되면 kubelet은 forcible shutdown을 트리거한다. container runtime은 container의 프로세스에 SIGKILL signal을 전송한다. 또한 kubelet은 pause container가 있는 container runtime에 대해 해당 container를 정리한다.
5. kubelet은 grace period를 0으로 설정해 API server에서 po를 force deletion한다.
6. API server는 po의 API object를 삭제한다.

### Forced Pod termination
기본 삭제에 대한 grace period는 30초다. kubectl delete 명령어의 --grace-period flag를 사용해 변경할 수 있다.

grace period를 0으로 설정하면 API server에서 po가 즉시 삭제된다. 만약 no에 po가 실행 중이라면 kubelet은 즉시 종료 프로세스를 시작한다.

**Note**: --force, --grace-period=0을 사용해 force deletion을 수행할 수 있다.

force deletion이 수행되면, API server는 no에서 po가 종료되었다는 kubelet의 확인을 기다리지 않는다. API server에서 즉시 po를 제거하므로 동일한 이름으로 새로운 po를 생성할 수 있다. 즉시 종료되도록 설정된 po는 강제 종료되기 전에 짧은 grace period가 제공된다.

### Garbage collection of failed Pods
실패한 po의 경우 API object는 명시적으로 po를 삭제할 때까지 API server에 존재한다.

control plane은 po의 수가 설정된 임계값(kube-controller-manager 내 terminated-pod-gc-threshold 값)을 초과할 때 종료된 po(Succeded 또는 Failed phase 포함)을 정리한다.