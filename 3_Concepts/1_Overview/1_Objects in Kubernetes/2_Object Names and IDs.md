cluster 내 각 object는 resource 유형 별로 고유한 name을 갖는다. 또한 모든 k8s object는 cluster 내에서 고유한 UID를 갖는다.

예를 들어 동일한 ns 내에서 myapp-1234 name을 갖는 1개의 po만 존재할 수 있다. 하지만 myapp-1234 name을 갖는 deploy는 존재할 수 있다.

사용자가 설정할 수 있는 속성인 label, annotation도 있다.

## Names
사용자는 resource URL에서 `/api/v1/pods/<name>`와 같이 object를 참조하는 문자열을 제공할 수 있다.

resource 내에서 해당 name은 1개의 object만 가질 수 있다. 물론 해당 object를 삭제하고 동일한 이름을 갖는 새로운 object를 생성할 수 있다.

name은 동일한 resource에 대해 모든 API 버전에서 유일해야 한다. API resource는 API group, resource type, namespace(namespaced resource일 경우)로 구분된다. 즉, API 버전은 상관이 없다.

> **Note**:  
> 물리 호스트를 나타내는 no와 같이 물리적 엔티티를 나타내는 object의 경우, no를 삭제한 후 다시 생성하지 않고 동일한 이름으로 호스트를 다시 생성하면, k8s는 새로운 호스트를 이전 호스트로 인식할 수도 있다.

아래는 resource 이름에 일반적으로 사용하는 네 가지 유형의 이름 제약 조건이다.

### DNS Subdomain Names

### RFC 1123 Label Names

### RFC 1035 Label Names

> **Note**:  
> RFC 1035, RFC 1123의 유일한 차이점은 RFC 1123은 숫자로 시작하는 것을 허용하지만 RFC 1035은 소문자 문자만 허용한다는 것이다.

### Path Segment Names
일부 k8s 리소스 타입의 경우 이름이 URL path로 사용되기 위해 인코딩이 필요할 수도 있다. 즉 이름이 ".", "..", "/", "%"를 포함하면 안될 수도 있다.

아래는 예시다.
``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-demo
spec:
  containers:
  - name: nginx
    image: nginx:1.14.2
    ports:
    - containerPort: 80
```
> **Note**:  
> 일부 리소스 타입의 경우 추가적인 제약사항이 있을 수 있다.

## UIDs
k8s 시스템은 object를 고유하게 구분하기 위해 시스템에서 생성하는 문자열을 UID로 사용한다.

모든 object는 k8s cluster의 전체 실행 기간 동안 고유한 UID를 갖는다. 이는 현재 시점 뿐만 아니라 과거 또는 현재에서 유사한 object의 생성을 서로 구분하기 위함이다.

k8s uid는 universally unique identifiers(UUID라고 부르기도 함)다. UUIDs are standardized as ISO/IEC 9834-8 and as ITU-T X.667.