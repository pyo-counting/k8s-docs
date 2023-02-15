kubelet 관리 container가 container 라이프사이클 훅 프레임워크를 사용해 관리 라이프사이클 동안 이벤트에 의해 트리거된 코드를 실행할 수 있는 방법에 대해 설명한다.

## Overview
k8s는 container에 lifecycle hook을 제공한다. hook을 통해 container는 lifecycle의 event를 인지할 수 있으며 이에 대응하는 lifecycle hook이 실행될 때 handler를 통해 구현한 코드를 실행할 수 있다.

## Container hooks
container에 노출되는 2개의 hook이 있다:

- PostStart: container 생성된 후 바로 실행되는 hook. 이 hook은 container ENTRYPOINT보다 먼저 실행된다는 보장은 없다. handler에 파라미터는 전달되지 않는다.
- PreStop: API 요청이나 liveness/startup probe 실패, preemption, resource contention 등의 management event로 인해 container가 종료되기 전에 바로 실행되는 hook. container가 이미 terminated 또는 completed 상태인 경우에는 PreStop hook 요청이 실패하며, hook은 container를 중지하기 위한 TERM signal이 보내지기 이전에 완료되어야 한다. po의 종료 grace period는 PreStop hook이 실행되기 전에 시작되어, handler의 결과에 상관없이 container가 po의 grace period 내에 결국 종료되도록 한다. handler에 파라미터는 전달되지 않는다.

### Hook handler implementations
container는 hook에 대한 handler를 구현(등록)함으로써 hook에 접근할 수 있다. container에서는 2가지 유형의 hook handler를 구현할 수 있다:

- Exec: container의 cgroup, namespace에서 pre-stop.sh와 같은 특정 명령어를 실행한다. 명령어를 통해 소모되는 컴퓨터 자원은 container내 자원에 포함된다.
- HTTP: container의 특정 endpoint를 대상으로 HTTP를 요청한다.

### Hook handler execution
container lifecycle 관리 hook이 호출되면, k8s 관리 시스템은 hook 액션에 따라 handler를 실행한다. httpGet, tcpSocket은 kubelet 프로세스에 의해 실행되며 exec는 container 내부에서 실행된다.

hook handler 호출은 container를 포함하는 po의 context 관점에서 동기적이다. PostStart hook의 경우, container의 ENTRYPOINT와 hook은 비동기적이다. 하지만 hook 실행이 너무 오래 걸리거나 hang에 걸린다면 container는 running 상태에 도달할 수 없다.

preStop hook은 container를 종료하기 위한 signal과 비동기적으로 실행되지 않는다. 그렇기 때문에 hook은 TERM signal이 전송되기 전에 실행을 완료해야 한다. 실행 중 PreStop hook이 hang에 걸리면 po의 phase가 Termination으로 변할 것이고 terminationGracePeriodSeconds가 만료된 후 po가 종료될 때 까지 남아있을 것이다. 이 grace period는 PreStop hook가 실행하는 데 걸리는 총 시간과 container가 정상적으로 중지하는 데 걸리는 총 시간에 적용된다. 예를 들어 terminationGracePeriodSeconds가 60이고 hook가 실행을 완료하는 데 55초가 걸리고 signal을 수신하고 정상적으로 종료하는 데 10초가 걸리면, container는 정상적으로 중지되기 전에 container가 죽을 것이다(terminationGracePeriodSeconds가 총 시간 55+10보다 짧기 때문).

PostStart 또는 PreStop hook이 실패하면 container를 kill한다.

사용자는 hook handler를 가능한 한 간단하게 만들어야 한다. 그러나 container를 종료하기 전에 상태를 저정할 때와 가티 긴 명령어 실행이 있을 수도 있다.

### Hook delivery guarantees

### Debugging Hook handlers