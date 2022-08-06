## Downward API
k8s와 결합되지 않으면서 container가 자신에 대한 정보를 갖는 것을 유용할 수 있다. downward API를 사용하면 k8s client 또는 API server를 사용하지 않고 클러스터 또는 자기 자신에 대한 정보를 이용할 수 있다.

container 내에 po, container 필드를 노출하는 2가지 방법이 있다.

- 환경 변수
- downwardAPI volume

이 두가 방식을 downward API라고 부른다.

## Available fields
downward API에 사용 가능한 필드를 설명한다.

### Information available via fieldRef

### Information available via resourceFieldRef

#### Fallback information for resource limits  
