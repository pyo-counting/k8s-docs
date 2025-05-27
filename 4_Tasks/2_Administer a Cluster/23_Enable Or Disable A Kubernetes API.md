이 페이지에서는 cluster의 control plane API 버전을 활성화하거나 비활성화하는 방법을 보여준다,

특정 API 버전은 kube-apiserver 명령어에 `--runtime-config=api/<version>` flag를 사용해 활성화하거나 비활성화할 수 있다. 이 flag의 값으로 쉼표로 구분된 API 버전 목록을 나열한다. 나중에 지정된 값이 이전에 지정된 값보다 우선 순위가 높다.

`runtime-config` flag는 다음과 같은 2가지 특수 key도 지원한다.
- `api/all`: known API를 나타낸다.
- `api/legacy`: legacy API만을 나타낸다. legacy API란 명시적으로 [deprecated](https://kubernetes.io/docs/reference/using-api/deprecation-policy/)된 모든 API를 의미한다.

예를 들어, v1을 제외한 모든 API 버전을 비활성화하려면 kube-apiserver에 `--runtime-config=api/all=false,api/v1=true`를 사용한다.