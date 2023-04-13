aggregation layer를 구성하면 core k8s API의 일부가 아닌 추가 API를 통해 k8s API server를 확장할 수 있다.

## Before you begin
**Note**: proxy와 extension apiserver간 상호 TLS 인증을 지원하는 환경에서 aggregation layer를 활성화하기 위해 몇 가지 설정 요구 사항이 있다. k8s와 kube-apiserver에는 여러 CA가 있으므로 proxy가 k8s 일반 CA 같은 것이 아니라 aggregation layer CA에 의해 서명되었는지 확인해야 한다.

**Caution**: 다른 클라이언트 타입을 위한 CA를 재사용하는 것은 클러스터 기능에 부정적인 영향을 미칠 수도 있다.

## Authentication Flow
crd와 다르게 aggregation API는 k8s 표준 apiserver에 추가적으로 사용자의 extension apiserver를 포함한다. k8s apiserver는 extension apiserver와 통신할 수 있어야하며 반대도 마찬가지다. 이러한 통신에 대한 보안을 위해 k8s apiserver는 X509 인증서를 사용해 extension apiserver에 대해 인증한다.

아래는 authentication, authorization 플로우에 대해 설명한다.

대략적인 플로우는 다음과 같디:

1. k8s apiserver: 요청 사용자를 인증(authentication)하고 요청된 API 경로에 대한 권한이 있는지 인가(authorization)한다.
2. k8s apiserver: 요청을 extension apiserver에 프록시한다.
3. extension apiserver: k8s apiserver의 요청을 인증한다.
4. extension apiserver: 요청에 대한 원래 사용자를 인가한다.
5. extension apiserver: 실행한다.

아래는 상세 다이어그램이다.
![diagram](https://d33wubrfki0l68.cloudfront.net/3c5428678a95c3715894011d8dd4812d2cf229b9/e745c/images/docs/aggregation-api-auth-flow.png)

### Kubernetes Apiserver Authentication and Authorization
extension apiserver에 의해 제공되는 API 경로에 대한 요청은 core k8s api 요청과 동일한 방법으로 시작한다: k8s apiserver에 API를 요청한다. 이 경로는 extension apiserver에 의해 k8s apiserver에 이미 등록되어 있다.

사용자는 k8s API 서버와 통신해 경로에 대한 접근을 요청한다. k8s apiserver는 표준 인증 및 인가를 사용해 사용자를 인증하고 해당 경로에 대한 접근을 인가한다.

위 단계는 표준 k8s API 요청과 동일하다.

인증 및 인가가 완료되면 k8s apiserver는 요청을 extension apiserver에 프록시할 준비가 완료됐다.

### Kubernetes Apiserver Proxies the Request
k8s apiserver는 이제 요청을 처리하기 위해 등록한 extension apiserver에 요청을 프록시한다. 이를 위해 몇가지 사항을 알아야 한다:

1. k8s apiserver는 네트워크를 통해 extension apiserver에 전달한 요청이 유효한 것임을 어떻게 인증할까?
2. k8s apiserver는 원래 요청이 인증된 사용자 이름과 그룹을 extension apiserver에 전달할 수 있을까?

위 두 가지를 위해 k8s apiserver에 몇 가지 플래그를 설정해야 한다.

#### Kubernetes Apiserver Client Authentication
k8s apiserver는 클라이언트 인증서를 통해 extension apiserver에 TLS 연결을 요청한다. 아래 플래그를 사용해 k8s apiserver를 초기화해야 한다:

- `--proxy-client-key-file`: private key file
- `--proxy-client-cert-file`: 클라이언트 인증서
- `--requestheader-client-ca-file`: 클라이언트 인증서를 서명한 CA 인증서
- `--requestheader-allowed-names`: 클라이언트 인증서 내 유효 CN(Common Name)

k8s apiserver는 `--proxy-client-*-file` 플래그로 지정한 파일을 이용해 extension apiserver에 인증한다. extension apiserver에서 요청을 유효한 것으로 간주하기 위해 아래 조건을 만족해야 한다:

1. `--requestheader-client-ca-file`: 연결은 이 플래그로 지정된 CA 인증서 파일을 통해 서명된 클라이언트 인증서를 사용해야 한다.
2. `--requestheader-allowed-names`: 연결은 이 플래그로 지정된 CN 목록에 대한 인증서를 사용해야 한다.

**Note**: `--requestheader-allowed-names`을 빈 값으로 설정하면 extension apiserver는 아무 CN을 허용한다.

위 옵션을 사용하면 k8s apiserver는:

1. extension apiserver에 인증하기 위해 옵션을 사용한다.
2. kube-system ns에 extension-apiserver-authentication cm을 생성해 CA 인증서와 허용 가능한 CN을 관리한다. 이 cm은 extension apiserver에서 요청에 대한 유효성 검사를 위해 사용한다.

k8s apiserver는 동일한 클라이언트 인증서를 사용해 모든 extension apiserver에 인증 요청한다. extension apiserver 별로 클라이언트를 생성하지 않고 k8s apiserver로 인증하기 위해 단일 인증서를 생성한다. 단일 인증서는 모든 extension apiserver 요청에서 재사용된다.

#### Original Request Username and Group
k8s apiserver가 extension apiserver에 요청을 프록시할 때 성공적으로 인증된 원래 요청의 사용자 이름 및 그룹을 프록시 요청의 HTTP header를 통해 제공한다. 사용할 header의 이름은 k8s apiserver에 설정해야 한다.

- `--requestheader-username-headers`: 사용자 이름
- `--requestheader-group-headers`: 그룹
- `--requestheader-extra-headers-prefix`: 추가 header 정보

위 header 이름은 extension-apiserver-authentication cm에도 존재하므로 extension apiserver에서 사용할 수 있다.

### Extension Apiserver Authenticates the Request
extension apiserver는 k8s apiserver에서 프록시된 요청을 수신하면 요청이 실제로 k8s apiserver가 수행하는 역할인 유효한 인증 프록시로부터 온 것인지 확인해야 한다. extension apiservser는 아래 과정을 통해 유효성을 검사한다:

1. kube-system ns 내에서 cm에서 아래 항목을 찾는다:
    - 클라이언트 CA 인증서
    - 허용된 CN 목록
    - 사용자이름, 그룹 및 기타 정보를 위한 HTTP header 이름
2. TLS 연결이 클라이언트 인증서에 대해 아래 항목을 확인한다:
    - cm 내 CA 인증서를 통해 서명됐는지 여부
    - cm 내 CN 목록(빈 값일 경우 모든 CN 허용)에 포함되는지 여부
    - header에서 사용자이름, 그룹을 추출한다

위 프로세스를 통과하면 요청은 유효한 인증 프록시(이 경우 k8s apiserver)로부터의 유효한 프록시 요청이다.

위는 extension apiserver 구현에 대한 책임이다. Many do it by default, leveraging the k8s.io/apiserver/ package. Others may provide options to override it using command-line options.

cm을 조회할 수 있는 권한을 가지기 위해 extension apiserver는 적절한 역할이 필요하다. 이를 위해 kube-system ns에 extension-apiserver-authentication-reader role이 있다.

### Extension Apiserver Authorizes the Request
extension apiserver는 header를 통해 추출한 사용자이름에 대해 프록시된 요청을 실행할 권한이 있는지 확인할 수 있다. 이는 k8s apiserver에 표준 SubjectAccessReview 요청을 전송해 수행한다.

extension apiserver가 k8s apiserver에 SubjectAccessReview 요청을 전송할 수 있는 권한을 스스로에게 부여하기 위해 올바른 권한이 필요하다. k8s는 적절한 권한이 있는 기본 system:auth-delegator ClusterRole이 있다. 이는 extension apiserver의 sa에 부여할 수 있다.

### Extension Apiserver Executes
SubjectAccessReview가 정상 요청되면 extension apiserver는 프록시된 요청을 처리한다.

## Enable Kubernetes Apiserver flags
아래 플래그를 사용해 kube-apiserver내 aggregation layer를 활성화 한다.

- `--requestheader-client-ca-file`: \<path to aggregator CA cert\>
- `--requestheader-allowed-names`: front-proxy-client
- `--requestheader-extra-headers-prefix`: X-Remote-Extra-
- `--requestheader-group-headers`: X-Remote-Group
- `--requestheader-username-headers`: X-Remote-User
- `--proxy-client-cert-file`: \<path to aggregator proxy cert\>
- `--proxy-client-key-file`: \<path to aggregator proxy key\>

> EKS 1.24 버전 API sever 로그 확인 시 기본적으로 위 flag가 설정된 것으로 확인된다. 

### CA Reusage and Conflicts

### Register APIService objects
어떤 클라이언트 요청이 extension apiserver에 프록시되어야 하는지 동적으로 설정할 수 있다. 아래는 설정 예시다:

``` yaml
apiVersion: apiregistration.k8s.io/v1
kind: APIService
metadata:
  name: <name of the registration object>
spec:
  group: <API group name this extension apiserver hosts>
  version: <API version this extension apiserver hosts>
  groupPriorityMinimum: <priority this APIService for this group, see API documentation>
  versionPriority: <prioritizes ordering of this version within a group, see API documentation>
  service:
    namespace: <namespace of the extension apiserver service>
    name: <name of the extension apiserver service>
  caBundle: <pem encoded ca cert that signs the server cert used by the webhook>
```

#### Contacting the extension apiserver
k8s apiserver가 요청을 extension apiserver로 프록시하기 위해서는 접근 방법을 알아야 한다.

`.spec.service` 필드는 extension apiserver에 대한 참조다. namespace, name은 필수 항목이다. 포트는 옵션이며 기본 값은 443이다.

Here is an example of an extension apiserver that is configured to be called on port "1234", and to verify the TLS connection against the ServerName my-service-name.my-service-namespace.svc using a custom CA bundle.

``` yaml
apiVersion: apiregistration.k8s.io/v1
kind: APIService
...
spec:
  ...
  service:
    namespace: my-service-namespace
    name: my-service-name
    port: 1234
  caBundle: "Ci0tLS0tQk...<base64-encoded PEM bundle>...tLS0K"
```