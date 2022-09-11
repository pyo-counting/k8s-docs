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

deploy는 업데이트 되는 동안 일정한 수의 po만 중단되는 것을 보장한다. 기본적으로 최소 75% 이상의 po가 동작하는 것을 보장한다.

또한 deploy는 최대 생성되는 po의 수를 제한한다. 기본적으로 최대 125% 이하의 po가 동작할 수 있도록 제한한다.

