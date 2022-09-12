ephemeral container는 트러블슈팅과 같이 사용자가 어떤 목적을 위해 동작 중인 po 내에서 일시적으로 실행하는 container다.

## Understanding ephemeral containers
po는 대체될 수 있고 언제든지 종료될 수 있기 때문에 이미 생성된 po 내에 container를 추가할 수 없다. 대신 deploy를 보통 사용해 제어하는 방식으로 po를 삭제하고 교체한다.

그러나 때때로 재현하기 어려운 버그의 문제 해결을 위해 기존 po의 상태를 검사해야 할 수 있다. 이 경우 사용자는 기존 po에서 ephemeral container를 실행해서 상태를 검사하고 명령어을 실행할 수 있다.

### What is an ephemeral container?
ephemeral container는 리소스 또는 실행에 대한 보증이 없다는 점에서 다른 container와 다르며 결코 자동으로 재시작되지 않는다. 그래서 애플리케이션을 만드는데 적합하지 않다. ephemeral container는 일반 container와 동일한 ContainerSpec 을 사용해서 명시하지만, 많은 필드가 호환되지 않으며 ephemeral container에는 허용되지 않는다.

- ephemeral container는 보통 포트를 갖지 않기 떄문에 ports, livenessProbe, readinessProbe 와 같은 필드는 허용되지 않는다.
- po에 할당된 리소스는 변경할 수 없으므로, resources 설정이 허용되지 않는다.
- 허용되는 필드의 전체 목록은 [EphemeralContainer reference documentation](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.25/#ephemeralcontainer-v1-core) 페이지를 확인한다.

ephemeral container는 pod.spec에 직접 추가하는 대신 API에서 특별한 `ephemeralcontainers` 핸들러를 사용해서 만들어지기 때문에 kubectl edit을 사용해서 ephemeral container를 추가할 수 없다.

일반 container와 마찬가지로, 사용자는 ephemeral container를 파드에 추가한 이후에 변경하거나 제거할 수 없다.

## Uses for ephemeral containers
ephemeral container는 container가 충돌 되거나 또는 container image에 디버깅 도구가 포함되지 않은 이유로 kubectl exec이 불충분할 때 문제 해결에 유용하다.

특히, distroless image를 사용하면 attack surface와 버그 및 취약점의 노출을 줄이는 최소한의 container image를 배포할 수 있다. distroless image는 shell 또는 어떤 디버깅 도구를 포함하지 않기 때문에, kubectl exec 만으로는 distroless image의 문제 해결이 어렵다.

ephemeral container 사용 시 프로세스 namespace 공유를 활성화하면 다른 container의 프로세스를 보는 데 도움이 된다.

