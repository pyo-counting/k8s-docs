## Understanding init containers
1개 이상의 init container는 app container가 실행되기 전에 먼저 실행된다. init container는 일반 container와 동일하지만 아래 예외가 있다:
- init container는 필요한 동작을 수행하고 정상 종료된다.
- 다음 init container가 실행되기 전에 성공 종료되어야 한다.

init container가 실패하면 kubelet은 성공할 때까지 재시작한다. 하지만 po의 `.spec.restartPolicy`가 Never일 때 init container가 실패하면 k8s는 po를 실패로 간주한다.

`.spec.containers`에 container를 작성하는 것과 같이 `.spec.initContainers`에 init container를 작성한다. 대부분의 동일한 필드를 갖는다.

init container의 status는 `.status.initContainerStatuses` 필드로 확인가능하다. 일반 container는 `.status.containerStatuses`

### Differences from regular containers
init container는 일반 app container와 동일하게 모든 필드를 사용할 수 있지만, resource request와 limit에 대해서는 일반 container와 다르게 다뤄진다.

init container는 lifecycle, livenessProbe, readinessProbe, startupProbe을 지원하지 않는다. 왜냐하면 po가 ready 상태가 되기 전에 실행이 완료되어야하기 때문이다. sidecar container에 대해서는 지원한다.

init container는 명시된 순서대로 실행되며, 각 init container는 이전 container가 실행 완료되면 순차적으로 실행된다. 실행이 모두 완료되면 kubelet은 app container를 초기화한다.

### Differences from sidecar containers
init container는 app container가 실행되기 전에 모두 실행된다. sidecar container는 app container와 같이 실행될 수 있다.

init container는 lifecycle, livenessProbe, readinessProbe, startupProbe을 지원하지 않지만 sidecar container에 대해서는 lifecycle을 제어하기 위해 모두 지원한다.

init container는 애플리케이션 container와 직접적으로 상호작용 하지는 않지만 resource를 공유한다. 두 container 종류는 데이터 교환을 위해 shared volume을 사용할 수 있다.

## Using init containers
init container는 app container와 다른 image를 사용하기 때문에 이점이 있다
- init container는 application container의 image에는 포함되지 않은 유틸리티나 커스텀 코드를 포함할 수 있다.
- The application image builder and deployer roles can work independently without the need to jointly build a single app image.
- init container는 동일 po내에서 app container와는 다른 파일시스템를 다룬다. 그렇기 때문에 app container에서 접근하지 못하는 secret에 접근할 수도 있다.
- init container는 app container가 실행되기 전에 실행이 완료되어야 한다. 이런 특성을 활용해 init container는 app container의 실행에 대한 선행 조건을 체크해 실행을 막을 수 있다.
- Init containers can securely run utilities or custom code that would otherwise make an app container image less secure. By keeping unnecessary tools separate you can limit the attack surface of your app container image.

### Examples

#### Init containers in use

## Detailed behavior
po의 시작 동안 kubelet은 networking, storage가 준비될 때까지 init container의 시작을 지연한다.

각 init container는 po의 `.spec`에 명시된 순서대로 시작되며 이전 init container가 성공적으로 종료되어야 다음이 실행된다. init container가 실패로 종료되면 restartPolicy를 따른다. 하지만 po의 restartPolicy가 Always라면 init container는 OnFailure로 간주된다.

init container가 성공할 때까지 po는 Ready 상태가 될 수 없다. init container의 port는 svc에 사용될 수 없다. 초기화 중인 po는 Pending status이며 Initialized condition은 false다.

po가 재시작되면 모든 init container도 재시작된다.

init container sepc에 대한 수정은 image로 제한된다. 이는 po를 재시작하는 것과 같다.

init container는 재시작될 수도 있기 때문에 init container의 동작은 멱등성을 가져야 한다. 특히 EmptyDirs에 파일 쓰기 작업을하는 코드는 이미 존재하는 파일에 대한 가능성을 염두해야 한다.

init container는 app container의 모든 필드를 사용할 수 있다. 하지만 init container는 container의 목표가 특정 동작을 수행 및 종료된다는 특성을 가지고 있기 때문에 k8s는 `readinessProbe`를 제한한다.

init container가 무한히 실패하는 상황에 빠지지 않도록 po에 `.spec.activeDeadlineSeconds`를 사용해야 한다. active deadline은 init container를 포함한다. 하지만 job으로 배포할 경우에만 사용하는 것을 권장한다. 왜냐하면 이 필드는 init container의 종료 후에도 영향을 미치기 때문이다. 즉 정상 동작하는 po가 이 설정 값의 영향으로 종료될 수도 있다.

init container, app container는 po 내에서 유일한 이름을 가져야한다.

### Resources sharing within containers
Given the order of execution for init, sidecar and app containers, the following rules for resource usage apply:
- The highest of any particular resource request or limit defined on all init containers is the effective init request/limit. If any resource has no resource limit specified this is considered as the highest limit.
- The Pod's effective request/limit for a resource is the higher of:
    - the sum of all app containers request/limit for a resource
    - the effective init request/limit for a resource
- Scheduling is done based on effective requests/limits, which means init containers can reserve resources for initialization that are not used during the life of the Pod.
- The QoS (quality of service) tier of the Pod's effective QoS tier is the QoS tier for init containers and app containers alike.
Quota and limits are applied based on the effective Pod request and limit.

### Init containers and Linux cgroups

### Pod restart reasons
po는 아래와 같은 사유로 재시작될 수 있다.
- po infrastructure container가 재시작. 이는 일반적이지 않으며, 보통 no에 대해 root 접근 권한을 가진 누군가에 의해 수행됐을 것이다.
- po의 모든 container가 종료됐으며 restartpolicy가 Always. init container의 완료에 대한 기록이 gc로 인해 삭제

init container의 image가 변경되거나, init container 실행 완료 기록이 gc로 인해 삭제되더라도 po는 재시작되지 않는다. 이는 k8s v1.20이후 적용된다.