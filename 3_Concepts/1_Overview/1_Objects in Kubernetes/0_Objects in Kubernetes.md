## Understanding Kubernetes objects
k8s object는 k8s 시스템에서 persistent entity이다. k8s는 이러한 entity를 사용하여 cluster의 상태를 나타낸다. 구체적으로 object는 다음을 나타낼 수 있다.
- 어떤 container화된 애플리케이션이 실행 중인지(그리고 어느 no에서 실행 중인지)
- 해당 애플리케이션에 대한 사용 가능한 resource
- 애플리케이션의 동작에 대한 policy. 예를 들어 다시 restart policy, upgrade, fault-tolerance

k8s object는 "의도의 기록(record of intent)"다. 즉, object를 생성하면 k8s 시스템이 해당 object가 존재하도록 지속적으로 관리한다. 사용자는 object를 생성함으로써 cluster 작업이 어떻게 보이길 원하는지 k8s 시스템에게 알려주는 것이며 이것이 cluster의 desired state다.

k8s object를 사용하려면(생성, 수정 또는 삭제) k8s API를 사용해야 한다. 예를 들어 kubectl CLI를 사용하면 필요한 [Kubernetes API](https://kubernetes.io/docs/concepts/overview/kubernetes-api/) 호출을 대신 수행한다. 또한 [Client Libraries](https://kubernetes.io/docs/reference/using-api/client-libraries/) 중 하나를 사용하여 직접 프로그래밍을 수행해 k8s API를 사용할 수도 있다.

### Object spec and status
거의 모든 k8s object에는 두 개의 중첩된 object 필드를 포함한다: object `spec`,과 `status`다. spec를 갖는 object의 경우, object를 생성할 때 이를 설정해야 하며, 이는 resource가 가져야 하는 특성에 대한 설명을 제공하며 desired state를 나타낸다.

status는 object의 current state를 설명하며 k8s 시스템과 구성 요소에 의해 제공, 업데이트된다. k8s control plance 지속적으로 모든 object의 실제 state를 관리하여 desired state와 일치시키기 위해 노력한다.

예를 들어: k8s에서 deploy는 cluster에서 실행 중인 애플리케이션을 나타낼 수 있는 object다. deploy를 생성할 때 deploy spec를 사용해 애플리케이션이 3개의 replica를 실행하도록 설정할 수 있다. k8s 시스템은 deploy spec를 읽고 애플리케이션의 세 개의 인스턴스를 시작하여 status를 spec에 맞게 업데이트한다. 그 중 하나의 인스턴스가 실패하는 경우(status가 변경) k8s 시스템은 spec와 status의 차이에 대응한다. 이 경우에는 대체 인스턴스를 시작해 3개의 replica를 다시 유지한다.

object의 spec, status, metadata에 대한 자세한 정보는 [Kubernetes API Conventions](https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md)를 참고한다.

### Describing a Kubernetes object
k8s에서 object를 생성할 때 desired state를 설명하는 object spec과 몇 가지 기본 정보(예를 들어 이름)를 설정해야 한다. k8s API를 사용하여 object를 생성할 때(직접 또는 kubectl을 통해), 해당 API request은 request body에 JSON 형식으로 이 정보를 포함해야 한다. 대부분의 경우 manifest라고 하는 파일을 사용하며 kubectl 명령어를 사용한다. 관례적으로, manifest는 YAML로 작성된다(또는 JSON 형식을 사용할 수도 있다). kubectl과 같은 도구는 HTTP를 통해 API 요청을 수행할 때 manifest 정보를 JSON 또는 다른 지원되는 serialization 형식으로 변환한다.

아래는 k8s deploy의 필수 필드, object spec을 나타내는 manifest 예시다.
``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2 # tells deployment to run 2 pods matching the template
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
manifest를 이용한 deploy 생성은 kubectl apply 명령어의 argument로 .yaml 파일을 사용하는 것이다.
``` sh
kubectl apply -f https://k8s.io/examples/application/deployment.yaml
```

명령어 결과는 다음과 같다.

``` sh
deployment.apps/nginx-deployment created

```

### Required fields
k8s object 생성을 위한 manifest(YAML 또는 JSON) 파일을 작성할 때 아래 필드는 설정해야 한다.
- `apiVersion`: object를 생성하는 데 사용하는 k8s API의 버전
- `kind`: 생성하려는 object의 유형
- `metadata`: object를 고유하게 식별하는 데 도움이 되는 데이터. 이름 문자열, UID, 선택적으로 ns를 포함한다.
- `spec`: k8s의 desired state

object spec의 정확한 형식은 모든 k8s object마다 다르며 해당 object마다 특별한 nested 필드가 있을 수 있다. [Kubernetes API reference](https://kubernetes.io/docs/reference/kubernetes-api/)는 k8s를 사용하여 생성할 수 있는 모든 object의 spec 형식을 찾는 데 도움이 될 수 있다.

예를 들어 po API 레퍼런스의 spec 필드를 확인해본다. 각 po에 대해 .spec desired state(예: po 내의 각 container image)를 지정한다. 또 다른 예는 sts API의 spec 필드를 확인해본다. sts에 대해 .spec 필드는 desired state를 나타낸다. sts의 .spec 내에는 po object에 대한 template도 포함된다. 해당 template sts controller가 sts specification을 충족하기 위해 생성할 po를 나타낸다. 서로 다른 종류의 object도 서로 다른 .status를 가질 수 있다. API reference 페이지에서 해당 .status 필드의 구조, 각각의 다른 유형의 object에 대한 내용이 자세히 설명되어 있다.

> **Note**:  
> YAML 설정 파일에 대한 추가 정보는 [Configuration Best Practices](https://kubernetes.io/docs/concepts/configuration/overview/)를 참고한다.

## Server side field validation
k8s v1.25부터 kube-apiserver는 object에서 인식되지 않거나 중복된 필드를 감지하는 서버 측 [field validation](https://kubernetes.io/docs/reference/using-api/api-concepts/#field-validation)을 제공한다. 이는 kubectl --validate의 모든 기능을 서버 측에서 제공한다.

kubectl 도구는 --validate flag를 사용해 field validation level을 설정한다. ignore, warn, strict 값을 허용하며 true(strict와 동일)와 false(ignore와 동일) 값도 허용한다. kubectl의 기본 field validation level은 --validate=true이다.

- Strict: 엄격한 field validation. field validation 실패 시 오류 발생
- Warn: field validation가 수행되지만 오류는 요청 실패 대신 경고로 표시됨
- Ignore: 서버 측 field validation가 수행되지 않음

kubectl이 field validation를 지원하는 kube-apiserver에 연결할 수 없는 경우 클라이언트 측 field validation를 사용한다 k8s 1.27 이상 버전은 항상 field validation를 제공한다. 이전 k8s 버전은 제공하지 않을 수 있다. cluster가 v1.27보다 오래된 경우 해당 k8s 버전 문서를 확인한다.