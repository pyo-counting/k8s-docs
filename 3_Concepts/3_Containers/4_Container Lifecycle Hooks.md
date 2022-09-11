## Overview
k8s는 container에 lifecycle hook을 제공한다. hook을 통해 container는 lifecycle의 event를 인지할 수 있으며 이에 대응하는 lifecycle hook이 실행될 때 handler를 통해 구현한 코드를 실행할 수 있다.

## Container hooks
container에 노출되는 2개의 hook이 있다:

- PostStart: container 생성된 후 바로 실행되는 hook. 이 hook은 container ENTRYPOINT보다 먼저 실행된다는 보장은 없다. handler에 파라미터는 전달되지 않는다.
- PreStop: API 요청이나 liveness/startup probe 실패, preemption, resource contention 등의 management event로 인해 container가 종료되기 전에 바로 실행되는 hook. container가 이미 terminated 또는 completed 상태인 경우에는 PreStop hook 요청이 실패하며, hook은 container를 중지하기 위한 TERM signal이 보내지기 이전에 완료되어야 한다. po의 종료 grace period는 PreStop hook이 실행되기 전에 시작되어, handler의 결과에 상관없이 container가 po의 grace period 내에 결국 종료되도록 한다. handler에 파라미터는 전달되지 않는다.

### Hook handler implementations
container는 hook의 handler를 구현 및 등록함으로써 해당 hook에 접근할 수 있다. container hook handler에는 2가지 타입이 있다.

- Exec: container의 cgroups와 namespace 안에서 pre-stop.sh와 같은 특정 커맨드를 실행. 커맨드에 의해 소비된 리소스는 해당 container에 계산된다.
- HTTP: container의 특정 엔드포인트에 대해서 HTTP 요청을 실행.

### Hook handler execution

### Hook delivery guarantees

### Debugging Hook handlers