
## Image names
container images는 `pause`, `example/mycontainer`, `kube-apiserver`와 같이 이름을 갖는다. 뿐만 아니라 `fictional.registry.example/imagename`과 같이 registry hostname을 포함할 수 있으며, 필요할 경우 `fictional.registry.example:10443/imagename`처럼 port도 명시할 수 있다.

registry hostname을 명시하지 않으면 k8s는 Docker public registry(hub.docker.com)로 간주한다. 기본 image registry는 [container runtime](https://kubernetes.io/docs/setup/production-environment/container-runtimes/) 설정을 통해 변경할 수 있다.

image 이름 뒤에 tag 또는 digest를 추가할 수 있으며, tag를 통해 동일한 image 중 버전을 구분한다. digest는 특정 버전의 고유 버전을 나타낸다. digest는 image의 내용에 대한 hash 값이며 immutable이다. tag는 변경할 수 있지만 digest는 그렇지 않다.

image tag는 대/소문자, 숫자, "\_", ".", "-" 문자 최대 128 글자로 구성된다. "\_", "-", "."와 같은 문자는 tag 내에서 구분자로 사용된다는 규칙이 있다. `[a-zA-Z0-9_][a-zA-Z0-9._-]{0,127}` 정규표현식에 match 돼야한다. 자세한 내용은 [OCI Distribution Specification](https://github.com/opencontainers/distribution-spec/blob/main/spec.md#workflow-categories)을 참고한다. tag를 명시하지 않으면 k8s는 `latest` tag로 가정한다.

image digest는 hash 알고리즘(예를 들어 sha256)을 이용해 생성된 hash 값을 갖는다. 예를 들어 `sha256:1ff6c18fbef2045af6b9c16bf034cc421a29027b800e4f9b68ae9b1cb3e9ae07`와 같은 값을 갖는다.

Some image name examples that Kubernetes can use are.
- busybox - Image name only, no tag or digest. Kubernetes will use Docker public registry and latest tag. (Same as docker.io/library/busybox:latest)
- busybox:1.32.0 - Image name with tag. Kubernetes will use Docker public registry. (Same as docker.io/library/busybox:1.32.0)
- registry.k8s.io/pause:latest - Image name with a custom registry and latest tag.
- registry.k8s.io/pause:3.5 - Image name with a custom registry and non-latest tag.
- registry.k8s.io/pause@sha256:1ff6c18fbef2045af6b9c16bf034cc421a29027b800e4f9b68ae9b1cb3e9ae07 - Image name with digest.
- registry.k8s.io/pause:3.5@sha256:1ff6c18fbef2045af6b9c16bf034cc421a29027b800e4f9b68ae9b1cb3e9ae07 - Image name with tag and digest. Only digest will be used for pulling.

## Updating images
deploy, sts, po와 같이 po template를 포함하는 object를 처음 생성할 때 기본적으로 명시하지 않은 경우 `imagePullPolicy`값은 IfNotPresent로 설정된다.

### Image pull policy
`imagePullPolicy`에 사용 가능한 값은 다음과 같다.
- IfNotPresent: 로컬에 image가 없는 경우에만 pull한다.
- Always: kubelet이 container를 실행할 때 마다 image digest를 얻기 위해 container image registry에 쿼리한다. kubelet은 로컬에 동일한 digest의 image가 있는 경우 해당 image를 사용하고 그렇지 않으면 pull한다.
- Never: kubelet은 image를 fetch하지 않는다. 로컬에 image가 있다면 해당 image를 사용하고, 없다면 실행에 실패한다.

The caching semantics of the underlying image provider make even imagePullPolicy: Always efficient, as long as the registry is reliably accessible. Your container runtime can notice that the image layers already exist on the node so that they don't need to be downloaded again.

> **Note**:  
> You should avoid using the :latest tag when deploying containers in production as it is harder to track which version of the image is running and more difficult to roll back properly.
>
> Instead, specify a meaningful tag such as v1.42.0 and/or a digest.

po가 항상 동일한 container image 버전을 사용하는 것을 보장하기 위해 `<image-name>:<tag>` 대신 `<image-name>@<digest>`를 사용할 수 있다.

po(와 po template)가 생성될 때 구동 중인 workload가 tag가 아닌 digest를 통해 정의되도록 조작해주는 third-pary admission controller가 있다. 이는 registery에서 tag가 변경되는 일이 발생해도 구동 중인 workload가 모두 같은 image를 사용하는 것을 보장하기 원하는 경우 유용하다.

#### Default image pull policy
kube-apiserver에 새로운 po를 제출할 때 cluster는 조건에 따라 `imagePullPolicy` 값을 자동 설정한다.
- `imagePullPolicy`를 생략, tag에 digest를 설정한 경우 IfNotPresent 값으로 설정
- `imagePullPolicy`를 생략, tag가 :latest일 때 Always 값으로 설정
- `imagePullPolicy`를 생략, tag를 명시하지 않을 경우 Always 값으로 설정
- `imagePullPolicy`를 생략, :latest가 아닌 tag일 때 IfNotPresent 값으로 설정

**Note:**
`imagePullPolicy`는 object가 처음 생성될 때 설정되며 tag가 업데이트 되더라도 변경되지 않는다.

#### Required image pull
항상 image를 pull하도록 강제하게 하도록 하기 위해 아래와 같이 설정할 수 있다.
- imagePullPolicy를 Always로 설정한다.
- imagePullPolicy를 생략하고 :latest tag를 사용한다. 이 경우 k8s는 Always로 설정한다.
- imagePullPolicy, tag를 생략한다. 이 경우 k8s는 Always로 설정한다.
- [AlwaysPullImages](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#alwayspullimages) admission controller를 사용한다.

### ImagePullBackOff
kubelet이 container runtime을 사용해 po의 container 생성을 시작할 때, `ImagePullBackOff`로 인해 container가 Waiting 상태에 있을 수 있다(po의 `.status.phase`가 Waiting).

ImagePullBackOff 상태는 k8s가 container image를 pull할 수 없어 실행할 수 없음을 의미한다. BackOff라는 단어는 k8s가 back-off 딜레이를 증가시키면서 image pulling을 계속 시도할 것임을 나타낸다.

k8s는 시간 간격을 늘리면서 계속 시도하며, 시간 간격의 상항은 k8s 코드에 5분으로 하드코딩 되어있다.

### Image pull per runtime class


## Serial and parallel image pulls
기본적으로 kubelet은 image pulling 작업을 병렬이 아닌 연속적으로 수행한다. 즉, image service에게 한 번에 하나의 image pull request만 요청하고 한 작업이 완료돼야 다른 작업을 수행한다.

no의 image pull 작업은 개별적으로 수행되기 때문에 다른 no와는 병렬적으로 동일한 image를 pull할 수 있다.

kubelet이 병렬로 image pull 수행을 처리하길 원한다면 `.serializeImagePulls`를 false로 설정한다. 추가적으로 `.maxParallelImagePulls`도 고려한다(기본 값은 nil로 제한이 없다).

kubelet에 병렬 image pull을 설정한 경우 contaienr runtime의 image service가 병렬 image pull을 다룰 수 있는지도 확인해야 한다.

kubelet은 1개의 po에 대해 여러 image를 병렬로 pull하지 않는다. 하지만 2개의 po가 서로 다른 image를 사용하는 경우에는 병렬로 image를 pull한다.

### Maximum parallel image pulls
`.serializeImagePulls`이 false인 경우 기본적으로 kubelet은 병렬 작업의 갯수에 제한을 두지 않는다. 제한을 두고 싶은 경우 kubelet의 `.maxParallelImagePulls` 필드를 사용한다.

이를 통해 네트워크 대역폭, disk I/O를 조절할 수 있다.

`.maxParallelImagePulls` 필드는 1 이상의 값이어야하며, 2 이상의 값을 사용하는 경우에는 `.serializeImagePulls` 필드가 false여야 한다. 그렇지 않으면 kubelet 실행에 실패한다.

## Multi-architecture images with image indexes
container registry는 바이너리 image 뿐만 아니라 container image index를 제공한다. image index는 container의 architecture 별 버전에 대한 여러 image manifest를 가리킬 수 있다. 그래서 컴퓨터 architecture에 적합한 binary image를 fetch할 수 있다.

k8s는 일반적으로 -${ARCH} 접미사를 붙여 container image 이름을 지정한다. 이전 버전과의 호환성을 위해 접미사가 있는 오래된 image를 생성해야 한다. The idea is to generate say pause image which has the manifest for all the arch(es) and say pause-amd64 which is backwards compatible for older configurations or YAML files which may have hard coded the images with suffixes.

## Using a private registry

### Configuring nodes to authenticate to a private registry

### Kubelet credential provider for authenticated image pulls

### Interpretation of config.json

### Pre-pulled images

### Specifying imagePullSecrets on a Pod

## Use cases

## Legacy built-in kubelet credential provider