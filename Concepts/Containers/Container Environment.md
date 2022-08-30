## Container environment
(...생략...)

### Container environment
k8s Container environment는 Container에 몇 가지 중요한 resource를 제공한다:

- 1개의 image와 1개 이상의 volume으로 조합된 파일시스템
- container 자체에 대한 정보
- cluster 내 다른 object에 대한 정보

#### Container information
container의 hostname은 container가 실행되는 po의 이름이다.

pd name, namespace는 downward API를 통해 환경 변수로 사용할 수 있다.

container image에 정의된 것 뿐만 아니라 pd 정의에서의 사용자 정의 환경 변수 역시 container에서 사용 가능하다.

#### Cluster information
container가 생성될 때 동일한 namespace에서 구동 중인 service들이 container 내 환경 변수로 제공된다.

bar라는 이름의 container에 매핑되는 foo라는 이름의 service에 대해서 다음의 형태로 환경 변수가 정의된다.

- FOO_SERVICE_HOST=\<the host the service is running on\>
- FOO_SERVICE_PORT=\<the port the service is running on\>