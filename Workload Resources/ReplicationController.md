- .spec.template.spec.restartPolicy은 기본값이 always이며 다른 값은 사용이 불가하다
- rc label(.metadata.labels)은 .spec.template.metadata.labels와 일반적으로 동일하게 설정하지만 다르게 설정할 수도 있다. 생략하면 .spec.template.metadata.labels와 동일하게 설정된다. 다른 값을 갖는다고 해서 동작이 변하지 않는다.
- .spec.selector는 label seletor로 rc가 관리하는 po를 지정한다. .spec.template.metadata.labels와 동일한 값을 가져야하며 다르면 api server가 거절한다. 생략하면 해당 값과 동일하게 설정된다.
- .spec.replicas 기본 값은 1이다
- `kubectl delete`로 rc를 삭제하는 것은 다음 단계를 순차적으로 진행한다.
  ```
  kubectl delete rc ${rc name}
  ```

  1. kubectl은 rc의 .spec.replicas를 0으로 scale한다.
  2. rc를 삭제하기 전에 각각의 po가 삭제되는 것을 기다린다.
  3. rc를 삭제한다.

- `kubectl delete` 명령어로 rc 삭제 시, `--cascade=orphan` flag를 사용해 rc resouce만 삭제할 수 있다(po는 삭제되지 않는다).
- .spec.template을 변경해 파트 템플릿을 변경해도 기존 생성된 파드가 삭제 및 다시 시작되지 않는다.
- rc가 관리하는 po의 label을 변경함으로써 rc의 관리에서 벗어날 수 있다. 이 때 rc는 .spec.replicas desire state를 위해 po를 추가 생성한다.
- 단일 svc 뒤에 여러 rc를 배치할 수있다. 이를 통해 트래픽을 기존 릴리즈와 새 릴리즈로 분산시킬 수 있다. svc뒤의 rc는 여러번 종료/시작이 될 수 있으며 이는 클라이언트 입장에서 알 수 없다(svc에서 로드밸런싱을 하기때문).
- rc에 의해 관리되는 po는 대체 가능(replacable)하며 의미적으로 동일하지만 시간이 지남에 따라 설정이 변경될 수 있다. 이는 stateless 서버에 매우 적합하다. ReplicationControllers can also be used to maintain availability of master-elected, sharded, and worker-pool applications.