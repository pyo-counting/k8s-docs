sts는 stateful 애플리케이션을 관리하기 위해 사용된다.

po 집합의 배포와 스케일링을 관리하며, po의 순서 및 고유성을 보장한다.

deploy와 유사하게 동일한 container spec을 기반으로 생성된 po를 관리하지만, deploy와 다르게 각 po의 고유성을 보장한다. po는 모두 동일한 sepc으로 생성됐지만 서로 대체 불가하다: 각 po는 스케줄링이 다시 될 때도 지속적으로 유지되는 식별자를 가진다.

## Using StatefulSets
아래 특성을 갖는 애플리케이션에 유용하다:
- Stable, unique network identifiers.
- Stable, persistent storage.
- Ordered, graceful deployment and scaling.
- Ordered, automated rolling updates.

위에서 stable은 po의 스케줄링에 따른 영속성(persistence)과 동일한 의미다. 애플리케이션이 고유 stable 식별자, 순서가 있는 배포, 삭제, 스케일링이 필요하지 않다면 다른 워크로드 리소스를 사용해야 한다. 이 경우 deploy, rs가 더 유용할 수 있다.

## Limitations
- 각 po의 storage는 요청된 sc을 기반으로 persistent volume provisoner에 의해 프로비저닝되거나 관리자에 의해 프로비저닝되어야 한다.
- sts에 대한 삭제 또는 scale down 시 관련된 volume은 삭제되지 않는다. 이는 일반적으로 sts와 연관된 것을 자동으로 제거하는 것보다 더 중요한 데이터의 안전을 보장하기 위함이다.
- sts는 현재 네트워크에서 po를 식별할 수 있도록 governing headless svc가 필요하다. 사용자는 이 svc를 생성할 책임이 있다.
- sts 삭제 시 po의 종료에 대해 어떠한 보장을 제공하지 않는다. sts에서는 po가 순차적이고 정상적으로 종료(graceful termination)되도록 하기 위해 삭제 전 sts의 스케일을 0으로 축소할 수 있다.
- 기본 pod management policy(OrderedReady)를 사용해 rolling update 시, broken 상태에 빠질 수 있으며 이를 해결하기 위해 수동 작업이 필요하다.

## Components
``` yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  selector:
    matchLabels:
      app: nginx # has to match .spec.template.metadata.labels
  serviceName: "nginx"
  replicas: 3 # by default is 1
  minReadySeconds: 10 # by default is 0
  template:
    metadata:
      labels:
        app: nginx # has to match .spec.selector.matchLabels
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: nginx
        image: registry.k8s.io/nginx-slim:0.8
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "my-storage-class"
      resources:
        requests:
          storage: 1Gi
```

- headless svc 대신 일반 ClusterIP 타입의 svc를 생성할 수도 있다.
- headless svc를 생성하면 각 po의 도메인에 대한 DNS SRV 레코드가 등록된다. svc에 대한 DNS lookup 시, 각 po에 대한 도메인 및 IP SRV, A 레코드가 반환된다.

### Pod Selector
`.spec.selector` 필드는 `.spec.template.metadata.labels` 필드와 매치되어야 한다. 이는 sts 생성시 검증한다.

### Volume Claim Templates
`.spec.volumeClaimTemplates` 필드는 pv provisioner에 의해 프로비저닝된 pv를 사용한다.

### Minimum ready seconds
`.spec.minReadySeconds`는 po가 '사용 가능(available)'이라고 간주될 수 있도록 po의 모든 container가 문제 없이 실행되어야 하는 최소 시간(초)을 설정하는 옵션 필드이다. 이는 rolling update를 사용할 떄 rollout의 진행 상황을 확인하는 데 사용된다. 이 필드의 기본값은 0이다(이 경우, po가 Ready 상태가 되면 바로 사용 가능하다고 간주된다).

## Pod Identity
sts po는 1) 순서, 2) 안정적인 네트워크 식별자, 3) 안정적인 스토리지로 구성되는 고유한 식별자 가진다. 식별자는 po가 어떤 no에 있는지 여부, 스케줄링과 상관없다.

### Ordinal Index 
N개의 레플리카가 있는 sts 내에서 각 po에 대해 0에서 N-1 까지의 정수가 순서대로 할당되며 해당 sts 내에서 고유하다.

### Stable Network ID
sts의 각 po는 sts의 이름과 po의 순번에서 hostname을 얻는다. hostname을 구성하는 패턴은 \$(statefulset name)-\$(ordinal) 이다. 위의 예시에서 생성된 3개 po의 이름은 web-0,web-1,web-2 이다. sts에 있는 po의 도메인을 제어하기 위해 headless svc를 사용할 수 있다. 이 svc가 관리하는 도메인은 \$(service name).\$(namespace).svc.cluster.local 의 형식을 가지며, 여기서 "cluster.local"은 클러스터 도메인이다. 각 po는 생성되면 \$(podname).\$(governing service domain) 형식을 가지는 DNS 서브 도메인을 갖게된다. 여기서 거버닝 서비스(governing service)는 sts의 `.spec.serviceName` 필드에 의해 정의된다.

클러스터에서 DNS가 구성된 방식에 따라, 새로 실행된 po의 DNS 이름을 즉시 찾지 못할 수 있다. 이 동작은 클러스터의 다른 클라이언트가 po가 생성되기 전에 po의 hostname에 대한 쿼리를 이미 보낸 경우에 발생할 수 있다. 네거티브 캐싱(DNS에서 일반적)은 이전에 실패한 조회 결과가 po가 실행된 후에도 적어도 몇 초 동안 기억되고 재사용됨을 의미한다.

po를 생성한 후 즉시 po를 검색해야 하는 경우, 몇 가지 옵션이 있다.

- DNS 조회에 의존하지 않고 쿠버네티스 API를 직접(예를 들어 watch 사용) 쿼리한다.
- k8s DNS 공급자의 캐싱 시간(일반적으로 CoreDNS의 cm을 편집하는 것을 의미하며, 현재 30초 동안 캐시함)을 줄인다.

사용자는 po의 네트워크 식별자에 대한 제공을 보장하기 위해 headless svc를 생성할 책임이 있다.

아래는 클러스터 도메인, svc 이름, sts 이름의 조합을 기반으로 sts po의 DNS이름에 어떻게 영향을 주는 예시를 보여준다.

| 클러스터 도메인 | svc(ns/name) | sts(ns\name) | sts domain | po DNS | po hostname |
|----------------|--------------|--------------|-----------|---------|------------|
| cluster.local | default/nginx | default/web | nginx.default.svc.cluster.local | web-{0..N-1}.nginx.default.svc.cluster.local | web-{0~N-1} |
| cluster.local | foo/nginx | foo/web | nginx.foo.svc.cluster.local | web-{0..N-1}.nginx.foo.svc.cluster.local | web-{0..N-1} |
| kube.local | foo/nginx | foo/web | nginx.foo.svc.kube.local | web-{0..N-1}.nginx.foo.svc.kube.local | web-{0..N-1} |

> Note: Note: Cluster Domain will be set to cluster.local unless [otherwise configured](https://v1-25.docs.kubernetes.io/docs/concepts/services-networking/dns-pod-service/).

### Stable Storage
sts에 정의된 VolumeClaimTemplate에 대해 각 po는 pvc를 전달 받는다. 위 예시에서 각 po는 my-storage-class 이름을 갖는 storage class와 1GiB의 프로비전된 storage를 가지는 pv를 전달 받는다. StorageClass를 명시하지 않으면 기본 값이 사용된다. po가 다시 스케줄링될 떄 po의 volumeMounts는 pvc와 관련된 pv가 마운트된다. sts이 삭제되더라도 pvc와 pv는 자동으로 삭제되지 않으며 수동으로 삭제해야 한다.

### Pod Name Label
sts controller가 po를 생성할 때, `statefulset.kubernetes.io/pod-name` label(값은 po의 이름)을 추가한다. 이 label을 이용해 sts의 특정 po에 svc를 연결할 수 있다.

## Deployment and Scaling Guarantees
- N개의 레플리카를 갖는 sts에 대해 po가 배포될 때 {0 ~ N-1} 순서로 순차적으로 생성된다.
- po가 삭제될 때 생성의 역순인 {N-1 ~ 0}순으로 삭제된다.
- po에 scale 작업을 적용하기 전에 모든 선행 po가 Running, Ready 상태여야 한다.
- n번 po가 종료되기 전에 n+1 po가 완전히 종료되어야 한다.

위는 `.spec.podManagementPolicy` 필드가 기본 값 OrderedReay 상태일 떄 동작 방식이다(scale up, po 교체, scale down 시 적용).

sts의 경우 `.spec.template.terminationGracePeriodSeconds` 필드에 대해 0 값을 사용하지 말아야 한다. 이는 안전하지 않으며 사용하지 않는 것을 권장한다. 관련해서 [force deleting StatefulSet Pods](https://v1-25.docs.kubernetes.io/docs/tasks/run-application/force-delete-stateful-set-pod/)을 참고한다.

위 nginx 예제에서 web-0, web-1, web-2 순서대로 po가 배포된다. web-1은 web-0이 Running, Ready 상태가 되기 전에 배포되지 않으며, web-2도 web-1에 대해 동일하다. web-1이 Running, Ready 상태가 된 이후 web-2이 배포되기 전에 web-0이 실패하면 web-2는 web-0이 성공적으로 다시 Running, Ready 상태가 되기 전에 배포되지 않는다.

sts의 replica는 1로 설정하면 web-2가 먼저 종료된다. web-1은 web-2이 완전히 종료 및 삭제되기 전까지 종료되지 않는다. web-2가 종료 및 삭제되고 web-1이 종료되기 전에 web-0이 종료되면 web-1은 web-0.이 성공적으로 다시 Ruinning, Ready 상태가 되기 전까지 종료되지 않는다.

### Pod Management Policies
`.spec.podManagementPolicy` 필드를 사용해 고유성 및 식별자를 유지하면서 순차 보증은 완화할 수 있다. 

#### OrderedReady Pod Management
`.spec.podManagementPolicy` 값이 OrderedReady. OrderedReady po management는 sts에 대해 기본 값이다.

#### Parallel Pod Management
`.spec.podManagementPolicy` 값이 Parallel. Parallel po management는 sts controller가 모든 po를 병렬로 배포/종료하도록 한다. 그리고 다른 po의 실행이나 종료에 앞서 po가 Running, Ready 상태가 되거나 완전히 종료되는 것을 기다리지 않는다. 이 옵션은 scaling 작업에 대한 동작에만 영향을 미치며 업데이트 작업에는 무관하다.

## Update strategies
sts의 `.spec.updateStrategy` 필드는 sts 내 po에 대한 container, label, 리소스 request/limit, annotation에 대한 자동화된 rolling update를 설정하거나 비활성화할 수 있다. 2가지 옵션이 있다.

- **OnDelete**: `.spec.updateStrategy.type`이 OnDelete일 경우 sts controller는 자동으로 sts의 po에 대해 업데이트하지 않는다. 사용자는 controller가 sts의 `.spec.template` 수정 사항을 적용하기 위한 po의 생성을 위해 수동으로 po를 삭제해야 한다.

- **RollingUpdate**: 기본 값으로 자동으로 po에 대한 rolling update를 수행한다.

## Rolling Updates
`.spec.updateStrategy.type`가 `RollingUpdate`일 경우, sts controller는 각 po를 삭제, 생성한다. 이는 순차적(N-1 ~ 0 순서)으로 po가 종료되고, 각 po의 업데이트는 한 번에 하나씩 진행된다.

k8s control plane은 이전 업데이트된 po가 Running, Ready 상태까지 기다리며 이후 po에 대해 업데이트를 수행한다. 만약 `.spec.minReadySeconds` 필드를 사용한다면 control plane은 추가적으로 po가 ready된 이후 해당 시간을 더 기다린다.

### Partitioned rolling updates
`.spec.updateStrategy.rollingUpdate.partition` 핃르를 사용해 RollingUpdate 업데이트 전략은 파티셔닝될 수 있다. 파티션을 명시하면 sts의 `.spec.template`이 업데이트 0 ~ N-1 값 중 파티션보다 크거나 같은 po에 대해서만 업데이트 된다. 파티션보다 작은 수를 가진 po는 업데이트 되지 않으며 삭제되더라도 이전 버전의 `.spec.template`을 사용해 재생성된다. 만약 sts의 `.spec.updateStrategy.rollingUpdate.partition` 이 `.spec.replicas` 보다 큰 경우 `.spec.template` 의 업데이트는 po에 대한 업데이트로 진행되지 않는다. 대부분의 경우 파티션을 사용할 필요가 없지만 업데이트를 준비하거나, canary roll out 또는 단계적인 roll out이 필요한 경우에는 유용하다.

### Maximum unavailable Pods
`.spec.updateStrategy.rollingUpdate.maxUnavailable` 필드를 사용해 업데이트 동안 사용 불가한 최대 po의 수를 제어할 수 있다. 정수 또는 퍼센트 값을 사용할 수 있다. 퍼센트 값을 적용한 값에 대해 올림해 정수 값으로 사용한다. 이 필드는 0일 수 없으며 기본 값은 1이다.

이 필드는 0 에서 replicas - 1 사이 범위에 있는 모든 po에 적용된다. 이 범위 내에 사용 불가능한 po가 있으면, maxUnavailable로 집계된다.

> Note: maxUnavailable 필드는 현재 alpha stage이며 MaxUnavailableStatefulSet feature gate가 활성화된 API 서버에서만 동작한다.

### Forced rollback
기본 po 관리 정책(OrderedReady)과 함께 rolling update를 사용할 경우 직접 수동으로 복구를 해야하는 실패 상태가 될 수 있다.

만약 po template을 Running, Ready 상태가 될 수 없는 설정으로 업데이트하는 경우(예시: 잘못된 바이너리 또는 애플리케이션 단 설정 오류로 ) sts은 roll out을 중지하고 대기한다.

이 상태에서는 po template을 올바른 설정으로 되돌리는 것으로 충분하지 않다. [known iusse](https://github.com/kubernetes/kubernetes/issues/67250)와 같이 sts는 손상된 po가 Ready(절대 되지 않음)될 때까지 기다리며 정상 동작하는 설정으로 되돌리는 것을 시도를 하기 전까지 기다린다.

template 되돌린 이후에는 추가적으로 sts이 잘못된 설정을 통해 생성 및 실행하려고 시도한 모든 po를 삭제해야 한다. 그러면 sts은 되돌린 template을 사용해서 po를 다시 생성하기 시작한다.

## PersistentVolumeClaim retention

### Replicas