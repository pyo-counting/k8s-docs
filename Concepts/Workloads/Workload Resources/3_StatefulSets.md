sts는 stateful 애플리케이션을 관리하기 위해 사용된다.

po 집합의 배포와 스케일링을 관리하며, po의 순서 및 고유성을 보장한다.

deploy와 유사하게 동일한 container spec을 기반으로 생성된 po를 관리하지만, deploy와 다르게 각 po의 고유성을 보장한다. po는 모두 동일한 sepc으로 생성됐지만 서로 대체 불가하다: 각 po는 스케줄링이 다시 될 때도 지속적으로 유지되는 식별자를 가진다.

## Using StatefulSets

## Limitations
- The storage for a given Pod must either be provisioned by a PersistentVolume Provisioner based on the requested storage class, or pre-provisioned by an admin.
- sts에 대한 삭제 또는 scale down 시 관련된 volume은 삭제되지 않는다. 이는 일반적으로 sts와 연관된 것을 자동으로 제거하는 것보다 더 중요한 데이터의 안전을 보장하기 위함이다.
- sts는 현재 po의 네트워크에서 po를 식별할 수 있도록 headless svc가 필요하다. 사용자는 이 svc를 생성할 책임이 있다.
- sts 삭제 시 po의 종료에 대해 어떠한 보증을 제공하지 않는다. sts에서는 파드가 순차적이고 정상적으로 종료(graceful termination)되도록 하기 위해 삭제 전 sts의 스케일을 0으로 축소할 수 있다.
- When using Rolling Updates with the default Pod Management Policy (OrderedReady), it's possible to get into a broken state that requires manual intervention to repair.

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

### Pod Selector
`.spec.selector` 필드는 `.spec.template.metadata.labels`필드와 매치되어야 한다. 이는 sts 생성시 검증한다.

### Volume Claim Templates
`.spec.volumeClaimTemplates` 필드는 pv provisioner에 의해 프로비저닝된 pv를 사용한다.

### Minimum ready seconds
`.spec.minReadySeconds`는 po가 '사용 가능(available)'이라고 간주될 수 있도록 po의 모든 container가 문제 없이 실행되어야 하는 최소 시간(초)을 설정하는 옵션 필드이다. 이는 rolling update를 사용할 떄 rollout의 진행 상황을 확인하는 데 사용된다. 이 필드의 기본값은 0이다(이 경우, po가 Ready 상태가 되면 바로 사용 가능하다고 간주된다).

## Pod Identity
sts po는 순서, 안정적인 네트워크 식별자, 안정적인 스토리지로 구성되는 고유한 식별자 가진다. 식별자는 po가 어떤 no에 있는지 여부, 스케줄링과 상관없다.

### Ordinal Index 
N개의 레플리카가 있는 sts 내에서 각 po에 대해 0에서 N-1 까지의 정수가 순서대로 할당되며 해당 sts 내에서 고유 하다.

### Stable Network ID
sts의 각 po는 sts의 이름과 po의 순번에서 hostname을 얻는다. hostname을 구성하는 패턴은 $(statefulset name)-$(ordinal) 이다. 위의 예시에서 생성된 3개 파드의 이름은 web-0,web-1,web-2 이다. sts에 있는 po의 도메인을 제어하기위해 headless svc를 사용할 수 있다. 이 svc가 관리하는 도메인은 $(service name).$(namespace).svc.cluster.local 의 형식을 가지며, 여기서 "cluster.local"은 클러스터 도메인이다. 각 po는 생성되면 $(podname).$(governing service domain) 형식을 가지고 일치되는 DNS 서브도메인을 가지며, 여기서 거버닝 서비스(governing service)는 스테이트풀셋의 serviceName 필드에 의해 정의된다.

### Stable Storage
sts에 정의된 VolumeClaimTemplate에 대해 각 po는 pvc를 전달 받는다. 위 예시에서 각 po는 my-storage-class 이름을 갖는 storage class와 1GiB의 프로비전된 storage를 가지는 pv를 전달 받는다. StorageClass를 명시하지 않으면 기본 값이 사용된다. po가 다시 스케줄링될 떄 po의 volumeMounts는 pvc와 관련된 pv가 마운트된다. sts이 삭제되더라도 pvc와 pv는 자동으로 삭제되지 않으며 수동으로 삭제해야 한다.

### Pod Name Label
sts controller가 po를 생성할 때, `statefulset.kubernetes.io/pod-name` label(값은 po의 이름)을 추가한다. 이 label을 이용해 sts의 특정 po에 svc를 연결할 수 있다.