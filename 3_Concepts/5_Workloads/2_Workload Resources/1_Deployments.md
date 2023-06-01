deploy는 po, rs에 대해 선언적 업데이트(declarative update)를 제공한다.

deploy에 desired state를 설정하면, deploy controller는 비율을 조정하며 현재 상태에서 desired state로 변경한다.

## Use Case
아래는 deploy를 사용하는 일반적 사례다:

- Create a Deployment to rollout a ReplicaSet. The ReplicaSet creates Pods in the background. Check the status of the rollout to see if it succeeds or not.
- Declare the new state of the Pods by updating the PodTemplateSpec of the Deployment. A new ReplicaSet is created and the Deployment manages moving the Pods from the old ReplicaSet to the new one at a controlled rate. Each new ReplicaSet updates the revision of the Deployment.
- Rollback to an earlier Deployment revision if the current state of the Deployment is not stable. Each rollback updates the revision of the Deployment.
- Scale up the Deployment to facilitate more load.
- Pause the rollout of a Deployment to apply multiple fixes to its PodTemplateSpec and then resume it to start a new rollout.
- Use the status of the Deployment as an indicator that a rollout has stuck.
- Clean up older ReplicaSets that you don't need anymore.

## Creating a Deployment
아래는 deploy 예시다. 이는 3개의 nginx po를 생성하기 위해 rs를 생성한다.

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

### Pod-template-hash label
**Catuon**: 이 label를 변경하면 안된다.

`pod-template-hash` label은 deploy가 생성 또는 관리하는 rs에 추가된다.

이 label은 deploy가 관리하는 여러 rs 간 구분을 위해 사용한다. rs의 `PodTemplate`를 해싱(hashing)한 결과를 label 값으로 사용한다.

## Updating a Deployment
**Note**: deploy의 rollout은 po template(`.spec.template`)이 변경되면 트리거된다.

deploy는 업데이트 되는 동안 일정한 수의 po만 중단되는 것을 보장한다. 기본적으로 최소 75% 이상의 po가 동작하는 것을 보장한다(`.spec.strategy.rollingUpdate.maxUnavailable`).

또한 deploy는 최대 생성되는 po의 수를 제한한다. 기본적으로 최대 125% 이하의 po가 동작할 수 있도록 제한한다(`.spec.strategy.rollingUpdate.maxSurge`).

**Note**: replicas - maxUnavailable <= availableReplicas <= replicas + maxSurge. availableReplicas는 terminating po를 포함하지 않는다. 결과적으로 rollout 시 deploy의 po는 예상한 것보다 많기 때문에 deploy에 의해 사용되는 리소스 역시 더 많다. 이러한 예상 밖의 사용량은 po의 terminationGracePeriodSeconds 시간이 지날때 까지 지속된다.

### Rollover (aka multiple updates in-flight)
deploy controller는 새로운 deploy을 발견하고, rs이 desired po를 생성하고 실행하는 것을 관찰한다. deploy의 .spec.template이 변경되면 새로운 rs가 생성되고 모든 기존 rs는 scale down된다.

만약 기존 rollout이 진행되는 중에 deploy를 업데이트하는 경우 deploy가 업데이트 마다 새 rs를 생성하고, scale up하기 시작한다. 그리고 scale up 작업 중이던 이전 rs를 rollover 한다 -- 히스토리 목록에 rs를 추가하고 scale down 진행한다.

예를 들어 nginx:1.14.2 image를 갖는 deploy reploca를 5로 설정 및 생성한다. nginx:1.14.2 replica가 3개 생성됐을 때 deploy를 업데이트해서 nginx:1.16.1 replica를 5개 생성성하도록 업데이트 한다고 가정한다. 이 경우 deploy는 즉시 생성된 3개의 nginx:1.14.2 po 3개를 종료하기 시작하고 nginx:1.16.1 po를 생성하기 시작한다. 즉 nginx:1.14.2 replioca가 5개가 생성되는 것을 기다리지 않는다.

### Label selector updates

**Note**: API 버전 apps/v1에서 deploy의 label selector는 생성 이후에 변경할 수 없다.

## Rolling Back a Deployment
기본적으로 deploy의 rollout 히스토리는 시스템에 저장되며 원하는 경우 언제든지 rollback수 있다(revision 히스토리 제한은 `.spec.revisionHistoryLimit` 필드를 통해 변경 가능).

**Note**: deploy의 revision은 deploy rollout이 트리거될 때 생성된다. 즉 deploy의 po template(.spec.template)이 변경될 때 생성된다. deploy의 replica 변경은 revision을 생성하지 않는다.

**Note**: deploy controller는 잘못된 rollout을 자동으로 중지하고, 새로운 rs의 scale up을 중지한다. 이는 rollingUpdage 파라미터인 maxUnavailable에 의해 따라 달라진다. 기본적으로 k8s는 이 값을 25%로 설정한다.

### Checking Rollout History of a Deployment

1. deploy의 revision history를 조회한다:

``` bash
kubectl rollout history deployment/nginx-deployment
```

출력:

```
deployments "nginx-deployment"
REVISION    CHANGE-CAUSE
1           kubectl apply --filename=https://k8s.io/examples/controllers/nginx-deployment.yaml
2           kubectl set image deployment/nginx-deployment nginx=nginx:1.16.1
3           kubectl set image deployment/nginx-deployment nginx=nginx:1.161
```

CHANGE-CAUSE 필드는 deploy의 kubernetes.io/change-cause annotation 값이다. CHANGE-CAUSE 값은 직접 수정할 수 있다:

- deploy에 annotation 설정하기: kubectl annotate deployment/nginx-deployment kubernetes.io/change-cause="image updated to 1.16.1"
- manifest 수정을 이용한 annotation 설정하기

2. --revsion flag를 사용해 특정 revision의 상세 내용을 확인할 수 있다.

``` bash
kubectl rollout history deployment/nginx-deployment --revision=2
```

출력:

```
deployments "nginx-deployment" revision 2
  Labels:       app=nginx
          pod-template-hash=1159050644
  Annotations:  kubernetes.io/change-cause=kubectl set image deployment/nginx-deployment nginx=nginx:1.16.1
  Containers:
   nginx:
    Image:      nginx:1.16.1
    Port:       80/TCP
     QoS Tier:
        cpu:      BestEffort
        memory:   BestEffort
    Environment Variables:      <none>
  No volumes.
```

### Rolling Back to a Previous Revision

1. 아래 명령어를 이용해 이전 revision으로 rollback한다

``` bash
kubectl rollout undo deployment/nginx-deployment
```

--to-revision flag를 사용해 특정 revision을 명시할 수도 있다.

이전 revision으로 rollback되면, revision에 대한 DeploymentRollback event가 deploy controller에 의해 생성된다.

## Scaling a Deployment

### Proportional scaling

## Pausing and Resuming a rollout of a Deployment
deploy 업데이트를 진행하거나 계획할 떄 deploy에 대한 rollout을 일시 중지할 수 있다. 뿐만 아니라 rollout을 다시 시작할 수 있다.

아래는 deploy에 대한 rollout을 한 번에 수행하기 위해 생성된 deploy에 대해 먼저 rollout을 중지한다.

- 아래 명령어를 사용해 rollout을 중지한다.

``` bash
kubectl rollout pause deployment/nginx-deployment
```

- 해당 deploy에 대해 여러 업데이트를 진행한다.

- 아래 명령어를 사용해 rollout을 다시 시작한다.

``` bash
kubectl rollout resume deployment/nginx-deployment
```

**Note**: 중지된 deploy를 다시 시작하기 전에 rollback을 할 수 없다.

## Deployment status
deploy는 라이플사이클 동안 여러 상태를 갖는다.

### Progressing Deployment
k8s는 아래 동작을 수행할 떄 deploy를 progressing으로 표시한다:

- deploy가 새로운 rs 생성
- deploy가 새로운 rs에 대해 scale up
- deploy가 이전 res에 대해 scale down
- 새로운 po가 Ready 또는 available (ready for at least MinReadySeconds)

rollout이 "progressing"이 될 때 deploy controller는 deploy의 .status.conditions에 아래 항목을 추가한다:

- type: Progressing
- status: "True"
- reason: NewReplicaSetCreated | reason: FoundNewReplicaSet | reason: ReplicaSetUpdated

kubectl rollout status 명령어를 사용해 deploy의 진행 상황을 모니터링할 수 있다.

### Complete Deployment
k8s는 아래와 같은 특성을 갖게되면 deploy를 complete로 표시한다:

- deploy과 관련된 모든 replica가 최신 버전으로 업데이트 되었을 때. 즉, 요청한 모든 업데이트가 완료되었을 때.
- deploy와 관련한 모든 replica를 사용할 수 있을 때
- deploy에 대해 이전 replica가 실행되고 있지 않을 때

rollout이 "complete"이 될 때, deploy controller는 deploy의 .status.conditions에 아래 항목을 추가한다:

- type: Progressing
- status: "True"
- reason: NewReplicaSetAvailable

이 Progressing condition은 새로운 rollout이 시작되기 전까지 "True" 상태값을 유지한다. replica의 가용성이 변경되는 경우에도(이 경우 Available condition에 영향을 미침) condition은 유지된다.

kubectl rollout status를 사용해서 deploy가 완료되었는지 확인할 수 있다. 만약 rollout이 성공적으로 완료되면 kubectl rollout status 는 종료 코드로 0이 반환된다.

### Failed Deployment
deploy는 새 rs 생성 및 배포 시 문제가 발생해 멈출 수 있다. 이는 아래와 같은 여러가지 요인으로 인해 발생한다:

- 할당량 부족
- readiness probe 실패
- image pull 에러
- 권한 부족
- limit range
- 애플리케이션 런타임의 잘못된 구성

이러한 condition을 디버깅할 수 있는 한 가지 방법은 deploy spec에서 데드라인 파라미터를 지정하는 것이다(.spec.progressDeadlineSeconds). .spec.progressDeadlineSeconds는 deploy의 배포가 정지됐음을 deploy의 status에 나타내기까지 deploy controller가 대기하는 시간을 나타낸다.

다음 kubectl 명령어로 progressDeadlineSeconds를 설정해서 controller가 10분 후 deploy rollout에 대한 진행 상태가 완료돼지 않았음에 대한 리포트를 수행하게 한다.

``` bash
kubectl patch deployment/nginx-deployment -p '{"spec":{"progressDeadlineSeconds":600}}'
```

만약 데드라인을 초과하면 deploy controller는 deploy의 .status.conditions에 다음의 deploy 컨디션(DeploymentCondition)을 추가한다:

- type: Progressing
- status: "False"
- reason: ProgressDeadlineExceeded

이 condition은 데드라인 보다 더 일찍 실패할 수도 있으며 이러한 경우 reason: ReplicaSetCreateError, status: "False"로 설정한다. deploy의 rollout이 완료되면 데드라인은 더 이상 고려되지 않는다.

status condition에 대한 자세한 내용은 [Kubernetes API conventions](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md#typical-status-properties)을 참고한다.

**Note**: k8s는 reason: ProgressDeadlineExceeded의 status condition을 보고하는 것 이외에 정지된 deploy에 대해 조치를 취하지 않는다. 더 높은 수준의 오케스트레이터는 이를 활용할 수 있다. 예를 들어를 deploy를 이전 버전으로 rollback할 수 있다.

**Note**: 만약 deploy rollout을 일시 중지하면 k8s는 지정된 데드라인과 비교하여 진행 상황을 확인하지 않는다. rollout 중에 deploy rollout을 안전하게 일시 중지하고, 데드라인을 넘기는 것과 관계없이 재개할 수 있다.

설정한 타임아웃이 짧거나 일시적으로 처리될 수 있는 다른 종료의 에러로 인해 deploy에 일시적인 에러가 발생할 수 있다. 예를 들어, 할당량이 부족하다고 가정해본다:

``` bash
kubectl describe deployment nginx-deployment
```

출력:

```
<...>
Conditions:
  Type            Status  Reason
  ----            ------  ------
  Available       True    MinimumReplicasAvailable
  Progressing     True    ReplicaSetUpdated
  ReplicaFailure  True    FailedCreate
<...>
```

kubectl get deployment nginx-deployment -o yaml 을 실행하면 deploy 상태는 다음과 유사하다.

``` yaml
status:
  availableReplicas: 2
  conditions:
  - lastTransitionTime: 2016-10-04T12:25:39Z
    lastUpdateTime: 2016-10-04T12:25:39Z
    message: Replica set "nginx-deployment-4262182780" is progressing.
    reason: ReplicaSetUpdated
    status: "True"
    type: Progressing
  - lastTransitionTime: 2016-10-04T12:25:42Z
    lastUpdateTime: 2016-10-04T12:25:42Z
    message: Deployment has minimum availability.
    reason: MinimumReplicasAvailable
    status: "True"
    type: Available
  - lastTransitionTime: 2016-10-04T12:25:39Z
    lastUpdateTime: 2016-10-04T12:25:39Z
    message: 'Error creating: pods "nginx-deployment-4262182780-" is forbidden: exceeded quota:
      object-counts, requested: pods=1, used: pods=3, limited: pods=2'
    reason: FailedCreate
    status: "True"
    type: ReplicaFailure
  observedGeneration: 3
  replicas: 2
  unavailableReplicas: 2
```

데드라인이 초과하면 k8s는 Progressing condition의 status, reason을 업데이트 한다:

```
Conditions:
  Type            Status  Reason
  ----            ------  ------
  Available       True    MinimumReplicasAvailable
  Progressing     False   ProgressDeadlineExceeded
  ReplicaFailure  True    FailedCreate
```

할당량 부족에 대한 문제를 해결한 후 deploy controller가 rollout을 완료하면 Progressing condition에 대해 status: "True" reason: NewReplicaSetAvailable로 업데이트 한다.

```
Conditions:
  Type          Status  Reason
  ----          ------  ------
  Available     True    MinimumReplicasAvailable
  Progressing   True    NewReplicaSetAvailable
```

type: Available, status: "True"는 deploy가 최소 가용성을 갖고 있음을 의미한다. 초소 가숑성은 deploy의 .spec.strategy 내 파라미터를 통해 설정 가능하다. type: Progressing, status: "True"은 deploy가 rollout 진행 중, rollout 완료, 최소 가용성 갖고 있음을 의미한다.

### Operating on a failed deployment
All actions that apply to a complete Deployment also apply to a failed Deployment. You can scale it up/down, roll back to a previous revision, or even pause it if you need to apply multiple tweaks in the Deployment Pod template.

## Clean up Policy
deploy 내 .spec.revisionHistoryLimit 필드를 사용해 유지할 이전 버전의 rs의 개수를 설정할 수 있다. 나머지는 백그라운드에서 ge된다. 기본 값은 10이다.

**Note**: 0으로 설정하면 deploy에 대해 rollback이 불가하다.

## Canary Deployment
deploy를 사용해서 일부 사용자 또는 서버에 릴리즈를 rollout하기 위해서는 [managing resources](https://v1-23.docs.kubernetes.io/docs/concepts/cluster-administration/manage-deployment/#canary-deployments)에 설명된 canary pattern에 따라 각 릴리즈 마다 여러 deploy를 생성할 수 있다.

## Writing a Deployment Spec

### Pod Template
.spec의 필수 필드는 .spec.template, .spec.selector다.

.spec.template은 po의 template이다. 

.spec.template.spec.restartPolicy은 Always 값만 허용되며, 기본 값이다.

### Replicas
.spec.replicas는 po의 desired 개수로 기본 값은 1이다.

deploy의 scale 관리를 위한 hpa가 있을 경우 .spec.replicas 필드를 설정하면 안된다.

대신 k8s control plane이 .spec.replicas 필드를 자동으로 관리하도록 한다.

### Selector
.spec.selector은 .spec.template.metadata.labels와 매칭돼야 다를 경우 API에 의해 거절된다.

API 버전 apps/v1 에서는 .spec.selector와 .metadata.labels이 설정되지 않으면 .spec.template.metadata.labels 은 기본 설정되지 않는다. 그렇기 떄문에 명시적으로 설정해야 한다. 또한 apps/v1에서는 deploy를 생성한 후 .spec.selector이 변경되지 않는 점을 주의한다.

deploy는 template의 .spec.template와 다르거나 po의 수가 .spec.replicas를 초과할 경우 selector와 일치하는 label을 가진 po를 종료할 수 있다. po의 수가 의도한 수보다 적을 경우 .spec.template을 이용해 새 po를 생성한다.

**Note**: slector와 매칭되는 label을 갖는 po를 추가적으로 생성하면 안된다. If you do so, the first Deployment thinks that it created these other Pods. Kubernetes does not stop you from doing this.

만약 selector가 겹치는 controller가 어러 개 있는 경우 controller 간 충돌로 인해 예상치 않은 동작을 할 수도 있다.

### Strategy
.spec.strategy는 po 교체에 사용되는 필드다. .spec.strategy.type 필드는 "Recreate" 또는 "RollingUpdate" 값을 가질 수 있다. 기본 값은 RollingUpdate다.

#### Recreate Deployment
새로운 po가 생성되기 전에 이전 po가 모두 삭제된다.

**Note**: 이렇게 하면 업그레이드를 생성하기 전에 모든 po 종료를 보장할 수 있다. deploy를 업그레이드하면, 이전 revision의 모든 po가 즉시 종료된다. 신규 revision의 po가 생성되기 전에 성공적으로 삭제가 완료되기를 대기한다. po를 수동으로 삭제하면, 라이프사이클은 rs에 의해 제어되며(이전 po가 여전히 terminating state에 있는 경우에도) 교체용 po가 즉시 생성된다. po에 대해 "최대" 보장이 필요한 경우 sts의 사용을 고려해야 한다.

#### Rolling Update Deployment
rolling update를 제어하기 위해 maxUnavailable, maxSurge 필드를 사용할 수 있다.

##### Max Unavailable
.spec.strategy.rollingUpdate.maxUnavailable는 업데이트 프로세스 동안 이용 불가한 po의 최대 개수를 설정하는 옵션 필드다. po의 개수를 나타내는 절대 값이나 퍼센트 값을 사용할 수 있다. 퍼센트는 절대값 계산 시 내림한다. .spec.strategy.rollingUpdate.maxSurge가 0이면 이 필드는 0이될 수 없다. 기본 값은 25%다.

30%로 설정한 경우, rolling update가 시작되면 이전 rs는 desired po의 수를 즉시 70%로 scale down한다. 새로운 po가 Ready되면 이전 rs는 추가적으로 scale down을 수행하며 이에 따라 새로운 rs는 추가적으로 scale up을 수행한다. 업데이트 동안 po의 총 개수는 항상 70%이상을 유지한다.

##### Max Surge
.spec.strategy.rollingUpdate.maxSurge는 업데이트 프로세스 동안 이용 가능한 po의 최대 개수를 설정하는 옵션 필드다. po의 개수를 나타내는 절대 값이나 퍼센트 값을 사용할 수 있다. 퍼센트는 절대값 계산 시 올림한다. .spec.strategy.rollingUpdate.maxUnavailable가 0이면 이 필드는 0이될 수 없다. 기본 값은 25%다.

30%로 설정한 경우, rolling update가 시작되면 po의 개수가 130%가 넘지않도록 새로운 rs를 즉시 scale up한다. 이전 po가 삭제되면 새로운 rs는 추가적으로 scale up을 수행함으로써 업데이트 동안 최대 130% po가 실행된다.

### Progress Deadline Seconds
.spec.progressDeadlineSeconds 옵션 필드는 deploy 의 condition 중 type: Progressing, status: "False". and reason: ProgressDeadlineExceeded으로 설정하기 전에 rollout을 기다리는 시간을 설정한다. 적어도 이 필드는 .spec.minReadySeconds보다 커야한다.

### Min Ready Seconds
.spec.minReadySeconds 옵션 필드는 새로 생성된 po가 avaliable로 간주되기 전까지 기다리는 시간을 지정한다. 기본 값은 0이다(po가 ready가 되면 즉시 available로 간주된다). po의 condition 중 ready는 po가 요청을 처리할 수 있는 상태를 의미하며, 이 경우 load balancing을 위해 ep 목록에 추가된다(readiness probe가 존재할 경우 probe가 성공해야 ready condition이 trur가 된다).

### Revision History Limit
deploy의 revision 히스토리는 rs에 저장된다.

.spec.revisionHistoryLimit rollback을 위해 유지하는 이전 rs의 개수를 나타내는 필드다. 이러한 rs는 etcd 내부의 리소스을 소모한다. 각 deploy revision의 설정은 rs에 저장된다; 그렇기 때문에 rs를 삭제하면 해당 revision으로 rollback할 수 없다. 기본 값은 10이다.

0으로 설정하면 새로운 deploy에 대해 rollback을 수행할 수 없다.

### Paused
.spec.paused는 deploy를 중지 또는 재개하기 위한 옵션 boolean 필드다. 중지 된 deploy와 중지 되지 않은 deploy의 유일한 차이점은 중지된 deploy는 PodTemplateSpec에 대한 변경 새 rollout을 트리거 하지 않는다. deploy는 생성시 기본적으로 중지되지 않는다.
