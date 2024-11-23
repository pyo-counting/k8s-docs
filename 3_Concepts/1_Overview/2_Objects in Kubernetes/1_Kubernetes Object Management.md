kubectl CLI는 k8s object 생성, 관리를 위한 몇 가지 다른 방식을 제공한다. kubectl에 대한 자세한 내용은 [](https://kubectl.docs.kubernetes.io/)을 참고한다.

## Management techniques
> **Warning**:  
> k8s object는 1가지 기술을 통해 관리해야 한다. 섞어서 사용하는 경우 정의되지 않는 결과를 초래할 수도 있다.

| Management technique             | Operates on          | Recommended environment | Supported writers | Learning curve |
|----------------------------------|----------------------|-------------------------|-------------------|----------------|
| Imperative commands              | Live objects         | Development projects    | 1+                | Lowest         |
| Imperative object configuration  | Individual files     | Production projects     | 1                 | Moderate       |
| Declarative object configuration | Directories of files | Production projects     | 1+                | Highest        |

## Imperative commands
명령어 cli을 사용할 때 사용자는 cluster의 실제 object를 대상으로 직접 작업을 수행한다. 사용자는 작업을 kubectl argument 또는 flag로 제공한다.

이는 일회성 작업을 실행할 때 권장하는 방법이다. 이는 실제 object에 직접 작업을 수행하기 때문에 history를 제공하지 않는다.

### Examples
deploy object를 생성해 nginx container를 실행한다.\
``` sh
kubectl create deployment nginx --image nginx
```

### Trade-offs
object 파일과 비교했을 때의 장단점은 다음과 같다.
- 명령어는 단일 동작 단어로 표현된다.
- 명령어는 클러스터에 변경 사항을 적용하는 데 단일 단계만 필요하다.

단점은 다음과 같다.
- 명령어는 change review 프로세스와 통합되지 않는다.
- 명령어는 변경과 연관된 audit trail을 제공하지 않는다.
- 명령어는 현재 object에 대한 내용만 제공하고 history를 알 수 없다.
- 명령어는 새로운 object를 생성하는 데 template을 제공하지 않는다.

## Imperative object configuration
### Examples
### Trade-offs
## Declarative object configuration
### Examples
### Trade-offs