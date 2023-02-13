
## Image names
container images는 pause, example/mycontainer, kube-apiserver와 같이 이름을 갖는다. 뿐만 아니라 fictional.registry.example/imagename과 같이 registry hostname을 포함할 수 있으며, 필요할 경우 port도 명시할 수 있다.

registry hostname을 명시하지 않으면 k8s는 Docker public registry(hub.docker.com)로 간주한다.

image 이름 뒤에 tag를 추가할 수 있으며, tag를 통해 동일한 image 중 버전을 구분한다.

image tag는 대/소문자, 숫자, "\_", ".", "-" 문자로 구성될 수 있다. "\_", "-", "."와 같은 문자는 tag 내에서 구분자로 사용된다는 규칙이 있다. tag를 명시하지 않으면 k8s는 `latest` tag로 가정한다.

## Updating images
deploy, sts, po와 같이 po template를 포함하는 object를 처음 생성할 때 기본적으로 명시하지 않은 경우 `imagePullPolicy`값은 IfNotPresent로 설정된다.

### Image pull policy
`imagePullPolicy`에 사용 가능한 값은 다음과 같다:

- IfNotPresent: 로컬에 image가 없는 경우에만 pull한다.
- Always: kubelet이 container를 실행할 때 마다 image digest를 얻기 위해 container image registry에 쿼리한다. kubelet은 로컬에 동일한 digest가 있는 경우 해당 image를 사용하고 그렇지 않으면 pull한다.
- Never: kubelet은 image를 fetch하지 않는다. 로컬에 image가 있다면 해당 image를 사용하고, 없다면 실행에 실패한다.

The caching semantics of the underlying image provider make even imagePullPolicy: Always efficient, as long as the registry is reliably accessible. Your container runtime can notice that the image layers already exist on the node so that they don't need to be downloaded again.

po가 항상 동일한 container image 버전을 사용하는 것을 보장하기 위해 \<image-name\>:\<tag\> 대신 \<image-name\>@\<digest\>를 사용할 수 있다.

po(와 po template)가 생성될 때 구동 중인 workload가 tag가 아닌 digest를 통해 정의되도록 조작해주는 third-pary admission controller가 있다. 이는 registery에서 tag가 변경되는 일이 발생해도 구동 중인 workload가 모두 같은 image를 사용하는 것을 보장하기 원하는 경우 유용하다.

#### Default image pull policy
API server에 새로운 po를 제출할 때 아래와 같은 특정 조건을 만족할 경우 `imagePullPolicy` 값을 설정한다:

- `imagePullPolicy`를 생략, tag가 :latest일 때 Always 값으로 설정
- `imagePullPolicy`를 생략, tag를 명시하지 않을 경우 Always 값으로 설정
- `imagePullPolicy`를 생략, :latest가 아닌 tag일 때 IfNotPresent 값으로 설정

**Note:**
`imagePullPolicy`는 object가 처음 생성될 때 설정되며 tag가 없데이트 되더라도 변경되지 않는다.

#### Required image pull
항상 image를 pull하도록 강제하게 하도록 하기 위해:

- imagePullPolicy를 Always로 설정한다.
- imagePullPolicy를 생략하고 :latest tag를 사용한다.
- imagePullPolicy, tag를 생략한다.
- AlwaysPullImages admission controller를 사용한다.

#### ImagePullBackOff
kubelet이 container runtime을 사용해 po의 container 생성을 시작할 때, `ImagePullBackOff`로 인해 container가 Waiting 상태에 있을 수 있다.

ImagePullBackOff 상태는 k8s가 container image를 pull할 수 없어 실행할 수 없음을 의미한다. BackOff라는 단어는 k8s가 back off 딜레이를 증가시키면서 image pulling을 계속 시도할 것임을 나타낸다.

k8s는 시간 간격을 늘리면서 계속 시도하며, 시간 간격의 상항은 k8s 코드에 5분으로 하드코딩 되어있다.

## Multi-architecture images with image indexes
container registry는 바이너리 image 뿐만 아니라 container image index를 제공한다. image index는 container의 architecture 별 버전에 대한 여러 image manifest를 가리킬 수 있다. 그래서 컴퓨터 architecture에 적합한 binary image를 fetch할 수 있다.

k8s는 일반적으로 -${ARCH} 접미사를 붙여 container image 이름을 지정한다. 이전 버전과의 호환성을 위해 접미사가 있는 오래된 image를 생성해야 한다. The idea is to generate say pause image which has the manifest for all the arch(es) and say pause-amd64 which is backwards compatible for older configurations or YAML files which may have hard coded the images with suffixes.

## Using a private registry

### Configuring nodes to authenticate to a private registry

### Interpretation of config.json

### Pre-pulled images

### Specifying imagePullSecrets on a Pod

## Use cases
