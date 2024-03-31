label은 object에 부착하는 key-value 쌍이다. label은 사용자에게 의미가 있고 관련성이 있는 object의 식별 속성을 지정하는 데 사용된다. 하지만 label은 핵심 시스템에 직접적인 의미를 함축하지는 않는다. label은 object의 부분 집합을 구성하고 선택하는 데 사용할 수 있다. label은 object를 생성할 때 부착할 수 있으며 이후 언제든지 추가하고 수정할 수 있다. 각 object는 일련의 key-value label을 가질 수 있다. 각 key는 object 내에서 고유해야 한다.
``` json
"metadata": {
  "labels": {
    "key1" : "value1",
    "key2" : "value2"
  }
}
```

label은 효율적인 쿼리와 모니터링을 가능하게 하며 UI, CLI에서 사용하기에 적합하다. 식별 정보가 아닌 정보는 annotation을 사용해 기록해야 한다.

## Motivation

label 예시.
- "release" : "stable", "release" : "canary"
- "environment" : "dev", "environment" : "qa", "environment" : "production"
- "tier" : "frontend", "tier" : "backend", "tier" : "cache"
- "partition" : "customerA", "partition" : "customerB"
- "track" : "daily", "track" : "weekly"

[commonly used labels](https://kubernetes.io/docs/concepts/overview/working-with-objects/common-labels/)는 일반적으로 사용되는 label로 사용자는 편의에 따라 언제든지 변경할 수 있다. 다만 object 내에서 label key는 고유해야한다.

## Syntax and character set
label은 key-value 쌍이다. 유효한 label은 `/` 문자로 구분된 optional prefix, name으로 구성된다. The name segment is required and must be 63 characters or less, beginning and ending with an alphanumeric character ([a-z0-9A-Z]) with dashes (-), underscores (_), dots (.), and alphanumerics between. The prefix is optional. If specified, the prefix must be a DNS subdomain: a series of DNS labels separated by dots (.), not longer than 253 characters in total, followed by a slash (/).

prefix를 생략하면 label은 사용자의 개인 것으로 간주된다. automated system components (e.g. kube-scheduler, kube-controller-manager, kube-apiserver, kubectl, other third-party automation)는 사용자 object에 prefix를 사용해 label을 추가해야한다.

`kubernetes.io/`, `k8s.io/` prefix는 k8s 구성요소에서 사용할 수 있도록 예약([reserved](https://kubernetes.io/docs/reference/labels-annotations-taints/))됐다.

유효한 label 값은:
- must be 63 characters or less (can be empty),
- unless empty, must begin and end with an alphanumeric character ([a-z0-9A-Z]),
- could contain dashes (-), underscores (_), dots (.), and alphanumerics between.

## Label selectors
name, UID와 다르게 label은 고유하지 않다. 즉 여러 object가 동일한 label을 가질 수도 있다.

label selector을 이용해 사용자는 object을 식별할 수 있다. label은 k8s에서 그룹의 기본이다.

API는 현재 equality-based, set-based라는 두 개의 selector 타입을 지원한다. label selector에는 콤마를 사용해 여러 요구 사항을 명시할 수 있다. 즉 AND 논리연산자의 역할을 한다.

The semantics of empty or non-specified selectors are dependent on the context, and API types that use selectors should document the validity and meaning of them.

> **Note**:  
> rs와 같은 일부 API 타입에서 두 인스턴스의 label selector는 ns 내에서 겹치지 않아야 한다. the controller can see that as conflicting instructions and fail to determine how many replicas should be present.

> **Caution**:  
> equality-based와 set-based 조건에는 OR 논리연산자가 없다.

### Equality-based requirement
equality-based, inequality-based는 label key와 value 필터링을 허용한다. 매칭 object는 추가 label을 가질 수 있지만 명시한 모든 label 조건은 반드시 만족해야 한다. `=`, `==`, `!=` 연산자가 있다.
```
environment = production
tier != frontend
```

### Set-based requirement
set-based는 value 집합에 대한 label key 필터링을 허용한다. `in`, `notin`, `exists` 연산자가 있다.
```
environment in (production, qa)
tier notin (frontend, backend)
partition
!partition
```

set-based는 equality-based와 같이 사용할 수 있다.

## API
### LIST and WATCH filtering
LIST, WATCH 작업 시 쿼리 파라미터를 사용해 반환되는 object를 필터링하기 위해 label selector를 지정할 수 있다.

- equality-based: ?labelSelector=environment%3Dproduction,tier%3Dfrontend
- set-based: ?labelSelector=environment+in+%28production%2Cqa%29%2Ctier+in+%28frontend%29

### Set references in API objects
svc, rc와 같은 k8s object는 po와 같은 resource 집합을 선택하기 위해 label selector를 사용한다.

#### Service and ReplicationController
svc, rc의 타켓 po 집합은 equality-based labele selector를 이용해 정의된다.

#### Resources that support set-based requirements
job, deploy, rs, ds와 같은 resource들은 equality-based 뿐만 아니라 set-based도 지원한다.
``` yaml
selector:
  matchLabels:
    component: redis
  matchExpressions:
    - { key: tier, operator: In, values: [cache] }
    - { key: environment, operator: NotIn, values: [dev] }
```

#### Selecting sets of nodes
label selector는 po를 스케쥴링 할 no 집합을 제한하기 위한 [node selection](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/)에도 사용할 수 있다.

## Using labels effectively
어떤 resource에든 단일 label을 사용할 수 있지만 이것이 항상 best practice는 아니다. resource 집합을 서로 구별하기 위해 여러 label을 사용해야 하는 시나리오가 보통이다.

예를 들어 서로 다른 애플리케이션이 app label에 대해 서로 다른 값을 사용할 수 있지만 [guestbook example](https://github.com/kubernetes/examples/tree/master/guestbook/)와 같은 multi-tier 애플리케이션은 각 tier을 구별해야 한다. 프론트엔드는 다음과 같은 레이블을 가질 수 있다.
``` yaml
labels:
  app: guestbook
  tier: frontend
```

while the Redis master and replica would have different tier labels, and perhaps even an additional role label:
``` yaml
labels:
  app: guestbook
  tier: backend
  role: master
```

``` yaml
labels:
  app: guestbook
  tier: backend
  role: replica
```

## Updating labels
생성한 resource에 대해 relabel을 수행해야 할 수도 있다. 이를 위해 kubectl label 명령어를 사용한다.