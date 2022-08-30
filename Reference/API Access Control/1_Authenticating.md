## Users in Kubernetes
모든 k8s 클러스터에는 2가지 범주의 사용자가 있다: 1) k8s가 관리하는 서비스 계정과, 2) 일반 사용자.

k8s에는 일반 사용자 계정을 나타내는 resource가 따로 없다. 즉, 일반 사용자는 API 호출을 통해 k8s 클러스터에 추가할 수 없다. API 호출을 통해 일반 사용자를 추가할 수 없더라도 클러스터의 CA로 서명한 유효한 인증서를 제시하는 사용자는 인증된 것으로 간주된다. 이 떄 k8s 인증서의 'subject' 필드 내 'commonName' 필드(예를 들어 "/CN=bob")을 사용자 이름으로 사용한다. RBAC에서는 사용자가 resource에 대한 특정 행위(action)을 수행할 권한이 있는지 확인한다.

이와 반대로 서비스 계정은 k8s API에서 관리하는 사용자다. 이는 sa(ServiceAccount) resource로 관리된다. sa는 특정 ns에 바인딩되며 API 서버가 자동으로 생성하거나 API 서버 호출을 통해 생성 가능하다. sa는 Secret resource로 저장된 자격 증명 집합에 연결된다. Secret은 k8s API와 통신할 수 있도록 po에 마운트된다.

API 요청은 1) 일반 사용자, 2) 서비스 계정, 3) 익명 요청으로 처리된다.

## Authentication strategies
k8s는 autehntication 플러그인을 통해 API 요청을 인증(authentication)하기 위해 1) 클라이언트 인증서, 2) bearer token, 3) authenticating proxy를 사용한다. HTTP 요청이 API 서버에 전달되면 플러그인은 다음 속성들을 요청과 연관지으려고 시도한다:

- Username: 사용자를 식별하는 문자열. 예를 들어, kube-admin, jane@example.com
- UID: username보다 더 일관되고 고유한 사용자를 식별하는 문자열
- Groups: 문자열 집합으로 각 문자열은 논리적 사용자 집합에서 사용자의 자격을 나타낸다. 예를 들어, system:masters, devops-team
- Extra fields: authorizer에서 유용할 수 있는 추가 정보를 포함하는 문자열

이는 authentication 시스템에서는 의미가 없으며, authorizer에서 의미를 갖는다.

여러 authentication 방법을 활성화할 수 있다. 일반적으로 최소 2가지 방법은 사용해야한다:

- sa를 위한 sa token
- 사용자 authentication을 위한 다른 방법

여러 authenticator 모듈이 활성화된 경우 인증을 성공한 모듈에서 추가 인증 절차는 필요하지 않으므로 생략한다. API 서버는 authenticator 실행에 대한 순서를 보장하지 않는다.

`system:authenticated` 그룹은 모든 인증된 사용자의 그룹에 포함된다.

Integrations with other authentication protocols (LDAP, SAML, Kerberos, alternate x509 schemes, etc) can be accomplished using an authenticating proxy or the authentication webhook.

### X509 Client Certs
### Static Token File
### Bootstrap Tokens
### Service Account Tokens
### OpenID Connect Tokens
### Webhook Token Authentication
### Authenticating Proxy
## Anonymous requests
## User impersonation
## client-go credential plugins
### Example use case
### Configuration
### Input and output formats