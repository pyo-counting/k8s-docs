kubelet이 관리하는 container가 container lifecycle hook 프레임워크를 사용해 lifecycle 동안 이벤트에 의해 트리거된 코드를 실행할 수 있는 방법에 대해 설명한다.

## Overview
k8s는 container에 lifecycle hook을 제공한다. hook을 통해 container는 lifecycle의 event를 인지할 수 있으며 이에 대응하는 lifecycle hook이 실행될 때 handler를 통해 구현한 코드를 실행할 수 있다.

## Container hooks
container에 노출되는 2개의 hook이 있다.
- `PostStart`: container 생성된 후 바로 실행되는 hook. 이 hook은 container ENTRYPOINT보다 먼저 실행된다는 보장은 없다. handler에 파라미터는 전달되지 않는다.
- `PreStop`: API 요청이나 liveness/startup probe 실패, preemption, resource contention 등의 management event로 인해 container가 종료되기 전에 바로 실행되는 hook. container가 이미 terminated 또는 completed 상태인 경우에는 PreStop hook 요청이 실패하며, hook은 container를 중지하기 위한 TERM(SIGTERM) signal이 보내지기 이전에 완료되어야 한다. po의 종료 grace period는 PreStop hook이 실행되기 전에 시작되어, handler의 결과에 상관없이 container가 po의 grace period 내에 결국 종료되도록 한다. handler에 파라미터는 전달되지 않는다.

### Hook handler implementations
container는 hook에 대한 handler를 구현(등록)함으로써 hook에 접근할 수 있다. container에서는 3가지 유형의 hook handler를 구현할 수 있다.
- Exec: container의 cgroup, namespace에서 pre-stop.sh와 같은 특정 명령어를 실행한다. 명령어를 통해 소모되는 컴퓨터 자원은 container내 자원에 포함된다.
- HTTP: container의 특정 endpoint를 대상으로 HTTP를 요청한다.
- Sleep: 지정된 시간 동안 container를 pause한다.

### Hook handler execution
container lifecycle management hook이 호출되면, k8s 관리 시스템은 hook 액션에 따라 handler를 실행한다. httpGet, tcpSocket(deprecated), sleep은 kubelet 프로세스에 의해 실행되며 exec는 container 내부에서 실행된다.

`PostStart` hook handler 호출은 container가 생성될 때 초기화되며 container의 ENTRYPOINT와 `PostStart` hook은 동시에 트리거된다. 하지만 hook 실행이 너무 오래 걸리거나 hang에 걸린다면 container는 running 상태에 도달할 수 없다.

`PreStop` hook은 container를 종료하기 위한 signal과 비동기적으로 실행되지 않는다. 그렇기 때문에 hook은 TERM(SIGTERM) signal이 전송되기 전에 실행을 완료해야 한다. 실행 중 PreStop hook이 hang에 걸리면 po의 phase가 Termination으로 변할 것이고 terminationGracePeriodSeconds가 만료된 후 po가 종료될 때 까지 남아있을 것이다. 이 grace period는 PreStop hook가 실행하는 데 걸리는 총 시간과 container가 정상적으로 중지하는 데 걸리는 총 시간에 적용된다. 예를 들어 terminationGracePeriodSeconds가 60이고 hook가 실행을 완료하는 데 55초가 걸리고 signal을 수신하고 정상적으로 종료하는 데 10초가 걸리면, container는 정상적으로 중지되기 전에 container가 죽을 것이다(terminationGracePeriodSeconds가 총 시간 55+10보다 짧기 때문).

PostStart 또는 PreStop hook이 실패하면 container를 kill한다.

사용자는 hook handler를 가능한 한 간단하게 만들어야 한다. 그러나 container를 종료하기 전에 상태를 저정할 때와 가티 긴 명령어 실행이 있을 수도 있다.

### Hook delivery guarantees
hook에 대한 전달은 최소 1번이어야 한다. 즉, PostStart, PreStop과 같은 특정 이벤트에 대해 hook은 여러번 호출될 수도 있다. 이를 올바르게 처리하는 것은 hook 구현에 달려있다.

일반적으로 한 번의 전달이 이루어진다. 예를 들어 HTTP hook handler receiver가 다운되어 트래픽을 수신할 수 없는 경우 재전송 시도는 없다. 하지만 때때로 이중 전달이 발생할 수도 있다. 예를 들어 hook을 보내는 도중 kubelet이 재시작되면 해당 kubelet이 hook을 재전송할 수도 있다.

### Debugging Hook handlers
hook handler의 로그는 po event로 노출되지 않는다. PostStart의 경우 `FailedPostStartHook` event, PreStop의 경우 `FailedPreStopHook` event로 노출된다. handler가 어떤 이유로 실패한다면 event를 브로드캐스트한다. 실패한 FailedPostStartHook event를 직접 생성하기 위해 아래 po에 대한 manifest 중 postStart.exec.command를 "badcommand"로 변경해 적용한다:
``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: lifecycle-demo
spec:
  containers:
  - name: lifecycle-demo-container
    image: nginx
    lifecycle:
      postStart:
        exec:
          command: ["/bin/sh", "-c", "echo Hello from the postStart handler > /usr/share/message"]
      preStop:
        exec:
          command: ["/bin/sh","-c","nginx -s quit; while killall -0 nginx; do sleep 1; done"]
```

아래는 위 po manifest를 사용했을 때 출력되는 event 내용이다:
```
Events:
  Type     Reason               Age              From               Message
  ----     ------               ----             ----               -------
  Normal   Scheduled            7s               default-scheduler  Successfully assigned default/lifecycle-demo to ip-XXX-XXX-XX-XX.us-east-2...
  Normal   Pulled               6s               kubelet            Successfully pulled image "nginx" in 229.604315ms
  Normal   Pulling              4s (x2 over 6s)  kubelet            Pulling image "nginx"
  Normal   Created              4s (x2 over 5s)  kubelet            Created container lifecycle-demo-container
  Normal   Started              4s (x2 over 5s)  kubelet            Started container lifecycle-demo-container
  Warning  FailedPostStartHook  4s (x2 over 5s)  kubelet            Exec lifecycle hook ([badcommand]) for Container "lifecycle-demo-container" in Pod "lifecycle-demo_default(30229739-9651-4e5a-9a32-a8f1688862db)" failed - error: command 'badcommand' exited with 126: , message: "OCI runtime exec failed: exec failed: container_linux.go:380: starting container process caused: exec: \"badcommand\": executable file not found in $PATH: unknown\r\n"
  Normal   Killing              4s (x2 over 5s)  kubelet            FailedPostStartHook
  Normal   Pulled               4s               kubelet            Successfully pulled image "nginx" in 215.66395ms
  Warning  BackOff              2s (x2 over 3s)  kubelet            Back-off restarting failed container
```