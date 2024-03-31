k8s annotation을 사용해 비식별 메타데이터를 object에 추가할 수 있다.

## Attaching metadata to objects
label, annotations을 사용해 k8s object에 메타데이터를 첨부할 수 있다. label은 object를 선택하거나 특정 조건을 만족하는 object를 필터링하는 데 사용될 조건으로 사용할 수 있다. 이와 대조적으로 annotation은 object를 선택하는 데 사용되지 않는다. annotation 메타데이터는 label에서 허용하지 않는 문자를 포함할 수 있다.

annotation은 label과 같이 key/value map이다.
``` json
"metadata": {
  "annotations": {
    "key1" : "value1",
    "key2" : "value2"
  }
}
```

> **Note**:  
> map의 key/value는 모두 문자열로 설정해야 한다.

아래는 annotation으로 기록될 수 있는 정보들에 대한 예시다:

- Fields managed by a declarative configuration layer. Attaching these fields as annotations distinguishes them from default values set by clients or servers, and from auto-generated fields and fields set by auto-sizing or auto-scaling systems.
- Build, release, or image information like timestamps, release IDs, git branch, PR numbers, image hashes, and registry address.
- Pointers to logging, monitoring, analytics, or audit repositories.
- Client library or tool information that can be used for debugging purposes: for example, name, version, and build information.
- User or tool/system provenance information, such as URLs of related objects from other ecosystem components.
- Lightweight rollout tool metadata: for example, config or checkpoints.
- Phone or pager numbers of persons responsible, or directory entries that specify where that information can be found, such as a team web site.
- Directives from the end-user to the implementations to modify behavior or engage non-standard features.

annotation을 사용하는 대신 외부 데이터베이스나 디렉터리에 정보를 저장할 수도 있다. 하지만 관리가 더 어렵다.

## Syntax and character set
annotation은 key/value 쌍이다. 유효한 annotation key는 두 세그먼트(접두사, 이름)로 구성된다. optional 접두사와 이름은 / 문자로 구분된다. The name segment is required and must be 63 characters or less, beginning and ending with an alphanumeric character ([a-z0-9A-Z]) with dashes (-), underscores (_), dots (.), and alphanumerics between. The prefix is optional. If specified, the prefix must be a DNS subdomain: a series of DNS labels separated by dots (.), not longer than 253 characters in total, followed by a slash (/).

접두사가 생략되면 annotation key는 개인의 것으로 간주된다. automated system 구성요소(kube-scheduler, kube-controller-manager, kube-apiserver, kubectl 등)는 접두사를 명시해야 한다.

`kubernetes.io/`, `k8s.io/` 접두사는 k8s core system에서 사용되도록 예약됐다.