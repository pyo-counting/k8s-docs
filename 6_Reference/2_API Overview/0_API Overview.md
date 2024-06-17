## API versioning

## API groups
[API groups](https://github.com/kubernetes/design-proposals-archive/blob/main/api-machinery/api-group.md)은 k8s API 확장을 쉽게 만들어준다. REST API path와 serialized object의 `apiVersion`에 API group이 명시된다.

k8s는 몇 가지 API groups이 있다.
- core group은 REST path `/api/v1`이다. core group은 `apiVersion` 필드내 명시하지 않는다. 예를 들어 `apiVersion: v1`
- name dgroup은 REST path `/apis/$GROUP_NAME/$VERSION`이고 `apiVersion: $GROUP_NAME/$VERSION`로 사용한다.

모든 API group은 [Kubernetes API reference](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.30)를 참고한다.

## Enabling or disabling API groups

## Persistence