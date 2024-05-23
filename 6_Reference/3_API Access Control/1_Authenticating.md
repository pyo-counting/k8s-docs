## Users in Kubernetes
모든 k8s cluster에는 2가지 범주의 사용자가 있다: k8s가 관리하는 sa와, 일반 사용자

일반적으로는 다음과 같은 방법으로 사용자를 관리할 수 있다.
- private key를 배포하는 관리자
- Keystone, Google 계정과 같은 user store
- username, password를 포함하는 파일

k8s에는 일반 사용자 계정을 나타내는 resource가 따로 없다. 즉, 일반 사용자는 API 호출을 통해 k8s cluster에 추가할 수 없다.

API 호출을 통해 일반 사용자를 추가할 수 없더라도 cluster의 CA로 서명한 유효한 인증서를 제시하는 사용자는 인증된 것으로 간주된다. 이 떄 k8s 인증서의 'subject' 필드 내 'commonName' 필드(예를 들어 "/CN=bob")을 사용자 이름으로 사용한다. RBAC에서는 사용자가 resource에 대한 특정 행위(action)을 수행할 권한이 있는지 확인한다. 자세한 내용은 [certificate request](https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests/#normal-user)을 참고한다.

이와 반대로 sa는 k8s API를 통해 관리하는 사용자다. 이는 sa(ServiceAccount) resource로 관리된다. sa는 특정 ns에 바인딩되며 kube-apiserver가 자동으로 생성하거나 kube-apiserver 호출을 통해 생성 가능하다. sa는 secret resource로 저장된 credential 집합에 연결된다. secret은 kube-apiserver와 통신할 수 있도록 po에 마운트된다.

API 요청은 일반 사용자, sa, anonymous user 중 하나로 처리된다. 즉, kube-apiserver로의 모든 요청은 인증되어야하거나 anonymous user로 간주되어야 한다.

## Authentication strategies
k8s는 autehntication 플러그인을 통해 API 요청을 인증(authentication)하기 위해 1) 클라이언트 인증서, 2) bearer token, 3) authenticating proxy를 사용한다. HTTP 요청이 kube-apiserver에 전달되면 플러그인은 다음 속성들을 요청과 연관지으려고 시도한다.
- `Username`: 사용자를 식별하는 문자열. 예를 들어, kube-admin, jane@example.com
- `UID`: username보다 더 일관되고 고유한 사용자를 식별하는 문자열
- `Groups`: 문자열 집합으로 각 문자열은 논리적 사용자 집합에서 사용자의 자격을 나타낸다. 예를 들어, system:masters, devops-team
- `Extra fields`: authorizer에서 유용할 수 있는 추가 정보를 포함하는 문자열

이는 authentication 시스템에서는 의미가 없으며, [authorizer](https://kubernetes.io/docs/reference/access-authn-authz/authorization/)에서 의미를 갖는다.

여러 authentication 방법을 활성화할 수 있다. 일반적으로 최소 2가지 방법은 사용해야한다:
- sa를 위한 sa token
- 사용자 authentication을 위한 다른 방법

여러 authenticator 모듈이 활성화 됐을 때, 어떤 모듈에서 인증을 성공하면 다른 모듈에서의 추가 인증 절차는 필요하지 않으며 생략된다. kube-apiserver는 authenticator 실행에 대한 순서를 보장하지 않는다.

`system:authenticated` 그룹은 모든 인증된 사용자의 그룹에 포함된다.

Integrations with other authentication protocols (LDAP, SAML, Kerberos, alternate x509 schemes, etc) can be accomplished using an authenticating proxy or the authentication webhook.

### X509 client certificates
클라이언트 인증서 인증은 kube-apiserver에 `--client-ca-file` flag를 사용해 활성화한다. 해당 flag를 통해 참조된 파일은 kube-apiserver에 요청되는 클라이언트 인증서의 유효성을 검사할 수 있도록 적어도 1개의 CA를 포함해야 한다. 만약 클라이언트 인증서가 유효하다면 Subject 필드 내 commonName 필드가 요청에 대한 사용자 이름으로 사용된다. k8s 1.4부터 클라이언트 인증서는 Subject 필드 내 organizationName 필드를 사용해 사용자 그룹을 명시할 수도 있다.

아래는 openssl 명령어를 사용해 인증서 서명을 요청하는 예시다:
``` sh
openssl req -new -key jbeda.pem -out jbeda-csr.pem -subj "/CN=jbeda/O=app1/O=app2"
```

위 명령어는 username이 "jbeda", group은 "app1", "app2"에 속하는 csr(certificate signing request)을 생성한다.

아래는 인증서의 Subjet 필드 예시다.
```
Subject: C=KR, ST=<st1:city w:st="on">SEOUL</ct1:city>, L=SEOGU, O=IBM, OU=Garage, CN=Hololy/emailAddress=hololy@mail.net
```

### Static token file
kube-apiserver는 `--token-auth-file` flag에 명시된 파일에서 bearer token을 읽는다. token은 무한정 지속되며 token 목록은 kube-apiserver를 재시작하지 않는 이상 변경할 수 없다.

token 파일은 최소 3개의 컬럼을 갖는 csv 파일이다. 컬럼은 다음과 같다: token, user name, user uid, (optional)group name

> **Note**:  
> 1개 이상의 그룹을 갖는 경우 dobule quoting이 필요하다. 아래는 예시다.
> ```
> token,user,uid,"group1,group2,group3"
> ```

#### Putting a bearer token in a request
http 클라이언트에서 bearer token authentication를 사용할 떄, kube-apiserver는 `Authorization: Bearer <token>` header를 예상한다. The bearer token must be a character sequence that can be put in an HTTP header value using no more than the encoding and quoting facilities of HTTP. 아래는 예시다
```
Authorization: Bearer 31ada4fd-adec-460c-809a-9e56ceb75269
```

### Bootstrap tokens
새로운 cluster를 위해 간소화된 bootstraping을 허용하기 위해 k8s는 bootstrap token이라고 불리는 동적으로 관리되는 
bearer token 타입을 포함한다. 이 token들은 `kube-system` ns에서 secret으로 저장되며 동적으로 관리, 생성된다. kube-controller-manager의 TokenCleaner는 만료된 bootstrap token을 삭제한다.

token은 `[a-z0-9]{6}.[a-z0-9]{16}` 포맷을 갖는다. 첫 번째 구성 요소는 Token ID, 두 번째 구성요소는 Token Secert이다. 사용자는 아래와 같이 HTTP header를 사용해 token을 명시한다.
```
Authorization: Bearer 781292.db7bc3a58fc5f07e
```

kube-apiserver의 `--enable-bootstrap-token-auth` flag를 사용해 bootstrap token authentication을 활성화할 수 있다. 그리고 kube-controller-manager의 `--controllers` flag에 TokenCleaner controller을 명시해 활성화활 수 있다(예를 들어 `--controllers=*,tokencleaner`). kubeadm을 사용하는 경우에는 자동으로 이 작업을 수행한다.

authenticator는 `system:bootstrap:<Token ID>`로 인증한다. 이는 `system:bootstrappers` 그룹에 속한다. 네이밍과 그룹은 bootstrap 이후 사용자가 이 token을 사용하는 것을 방지하기 위해 의도적으로 제한된다. 사용자 이름과 그룹은 bootstrap cluster를 지원하기 위한 적절한 authorization 정책을 작성하는 데 사용될 수 있다.

자세한 내용은 [Bootstrap Tokens](https://kubernetes.io/docs/reference/access-authn-authz/bootstrap-tokens/)를 참고한다.

### Service account tokens
sa는 자동으로 활성화된 authenticator이며 요청을 검증하기위해 서명된 bearer token을 사용한다. 플러그인은 2가지 optional flag를 갖는다.
- `--service-account-key-file`: File containing PEM-encoded x509 RSA or ECDSA private or public keys, used to verify ServiceAccount tokens. The specified file can contain multiple keys, and the flag can be specified multiple times with different files. If unspecified, --tls-private-key-file is used.
- `--service-account-lookup`: If enabled, tokens which are deleted from the API will be revoked.

sa는 일반적으로 kube-apiserver에서 자동으로 생성되며 ServiceAccount admission controller를 통해 cluster에서 실행되는 po와 연결된다. bearer token은 po 내 well-known 위치에 마운트되며 cluster 내 프로세스가 kube-apiserver와 통신할 수 있도록 한다. sa은 PodSpec의 serviceAccountName 필드를 사용해 명시적으로 po에서 사용할 sa를 지정할 수 있다.

> **Note**:  
> serviceAccountName는 자동으로 설정되기 때문에 보통 생략한다.

``` yaml
apiVersion: apps/v1 # this apiVersion is relevant as of Kubernetes 1.9
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: default
spec:
  replicas: 3
  template:
    metadata:
    # ...
    spec:
      serviceAccountName: bob-the-bot
      containers:
      - name: nginx
        image: nginx:1.14.2
```

sa bearer token은 cluster 외부에서 사용할 수 있으며 k8s kube-apiserver와 통신이 필요한 job에 대한 ID를 생성하는 데 사용할 수도 있다. sa를 직접 생성할 수 있다.
``` sh
kubectl create serviceaccount jenkins

serviceaccount/jenkins created
```

관련 token을 생성한다.
``` sh
kubectl create token jenkins

eyJhbGciOiJSUzI1NiIsImtp...
```

생성된 token은 서명된 JWT(Json Web Token)을 갖는다.

``` bash
kubectl get secret jenkins-token-1yvwg -o yaml
```

서명된 JWT는 지정된 sa으로 인증하기 위한 bearer token으로 사용할 수 있다. 일반적으로 이러한 token은 kube-apiserver에 대한 cluster 내 접근을 위해 po에 마운트되지만 cluster 외부에서도 사용할 수 있다.

sa는 사용자 이름 `system:serviceaccount:(NAMESPACE):(SERVICEACCOUNT)`로 인증하고 `system:serviceaccounts`, `system:serviceaccounts:(NAMESPACE)` 그룹에 할당된다.

> **Warning**:  
> Because service account tokens can also be stored in Secret API objects, any user with write access to Secrets can request a token, and any user with read access to those Secrets can authenticate as the service account. Be cautious when granting permissions to service accounts and read or write capabilities for Secrets.

### OpenID Connect Tokens
OpenID Connect는 Microsoft Entra ID, Salesforce, Google와 같은 OAuth2 provider에서 지원하는 프로토콜이다. 이 프로토콜의 주요 기능은 access token과 같이 반환되는 추가 필드인 [ID Token](https://openid.net/specs/openid-connect-core-1_0.html#IDToken)이다. 이 token은 서버에 의해 서명된 JWT다.

사용자를 식별하기 위해 authenticator는 OAuth2 [token response](https://openid.net/specs/openid-connect-core-1_0.html#TokenResponse)의 `id_token`을 bearer token으로 사용한다. 아래는 ID Token을 통한 요청 예시다.

![](../../image/openid-connect-tokens.PNG)
1. identity provider에 로그인한다.
2. identity provider는 `access_token`, `id_token`, `refresh_token`을 응답한다.
3. kubectl을 사용하는 경우 kubeconfig 또는 `--token` flag에 `id_token`을 사용한다.
4. kubectl은 kube-apiserver에 authentication하기 위해 HTTP Authorization header에 `id_token`을 사용한다.
5. kube-apiserver는 JWT 서명이 유효한지 확인한다.
6. `id_token`이 만료되지 않았는지 확인한다.
    - 만약 CEL 표현식이 AuthenticationConfiguration과 함께 설정됐다면 claim 및/또는 사용자 유효성 검사를 수행한다.
7. 사용자에 대한 authorization을 수행한다.
8. authorization 이후 kube-apiserver는 kubectl에 응답한다.
9. kubectl이 사용자에게 결과를 응답한다.

사용자를 인증하기 위한 모든 데이터가 `id_token`에 포함되기 때문에 k8s는 identity provider에 확인할 필요가 없다. 이와 같은 모델에서 모든 요청은 stateless하기 때문에 authentication을 위한 매우 확장성이 높은 솔루션을 제공하지만 몇 가지 과제가 있다.
1. k8s는 authentication 프로세스를 트리거하기 위한 "web interface"가 없다. k8s는 credential을 수집하기 위한 브라우저, interface가 없기 때문에 사용자는 먼저 identity provider에 직접 인증해야 한다.
2. `id_token`을 취소(revoke)할 수 없으며 인증서처럼 짧은 시간 동안(몇 분)만 유효해야 한다. 따라서 몇 분마다 새로운 token을 발급해야 하기 때문에 매우 번거롭다.
3. k8s 대시보드에 인증하기 위해 kubectl proxy 명령이나 `id_token`을 주입하는 reverse proxy를 사용해야 한다.

#### Configuring the API Server
##### Using flags
##### Authentication configuration from a file
##### Using kubectl

### Webhook Token Authentication
webhook authentication은 bearer token을 확인하기 위한 hook이다.
- `--authentication-token-webhook-config-file` flag는 remote webhook 서비스에 접근하는 방법을 설명하는 설정 파일을 설정한다.
- `--authentication-token-webhook-cache-ttl` flag는 authentication을 cache하는 기간을 설정한다. 기본 값은 2분이다.
- `--authentication-token-webhook-version` flag는 webhook와의 정보를 송수신하는데 사용할 `authentication.k8s.io/v1beta1` 또는 `authentication.k8s.io/v1` TokenReview object를 설정한다.

설정 파일은 [kubeconfig](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/) 포맷을 사용한다. `.clusters`는 remote 서비스를 참조하고 `.users`는 kube-apiserver webhook을 참조한다. 아래는 예시다.
``` yaml
# Kubernetes API version
apiVersion: v1
# kind of the API object
kind: Config
# clusters refers to the remote service.
clusters:
  - name: name-of-remote-authn-service
    cluster:
      certificate-authority: /path/to/ca.pem         # CA for verifying the remote service.
      server: https://authn.example.com/authenticate # URL of remote service to query. 'https' recommended for production.

# users refers to the API server's webhook configuration.
users:
  - name: name-of-api-server
    user:
      client-certificate: /path/to/cert.pem # cert for the webhook plugin to use
      client-key: /path/to/key.pem          # key matching the cert

# kubeconfig files require a context. Provide one for the API server.
current-context: webhook
contexts:
- context:
    cluster: name-of-remote-authn-service
    user: name-of-api-server
  name: webhook
```

사용자가 kube-apiserver에 bearer token을 사용해 인증을 시도하면, authentication webhook은 remote service에 token을 포함하는 JSON-serialized TokenReview object을 POST 요청한다.

### Authenticating Proxy
kube-apiserver는 `X-Remote-User`와 같은 HTTP 요청 header 값을 사용해 사용자를 식별할 수 있도록 설정할 수 있다. 이는 요청 header 값을 설정하는 authenticating proxy을 사용할 때 유용하다.
- --requestheader-username-headers Required, case-insensitive. Header names to check, in order, for the user identity. The first header containing a value is used as the username.
- --requestheader-group-headers 1.6+. Optional, case-insensitive. "X-Remote-Group" is suggested. Header names to check, in order, for the user's groups. All values in all specified headers are used as group names.
- --requestheader-extra-headers-prefix 1.6+. Optional, case-insensitive. "X-Remote-Extra-" is suggested. Header prefixes to look for to determine extra information about the user (typically used by the configured authorization plugin). Any headers beginning with any of the specified prefixes have the prefix removed. The remainder of the header name is lowercased and percent-decoded and becomes the extra key, and the header value is the extra value.

## Anonymous requests
## User impersonation
## client-go credential plugins
### Example use case
### Configuration
### Input and output formats
## API access to authentication information for a client