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

`system:authenticated` 그룹은 모든 인증된 사용자가 속하는 그룹이다.

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

사용자가 kube-apiserver에 bearer token을 사용해 인증을 시도하면, authentication webhook은 remote service에 token을 포함하는 JSON-serialized `TokenReview` object을 POST 요청한다.

webhoo API object는 다른 k8s API object와 동일하게 [versioning compativility rules](https://kubernetes.io/docs/concepts/overview/kubernetes-api/)이 적용된다. webhook을 구현할 때 요청의 올바른 deserialization를 보장하기 위해 요청의 apiVersion 필드를 확인해야 하며 요청과 동일한 버전의 `TokenReview` object로 응답해야 한다.


### Authenticating Proxy
kube-apiserver는 `X-Remote-User`와 같은 HTTP 요청 header 값을 사용해 사용자를 식별할 수 있도록 설정할 수 있다. 이는 요청 header 값을 설정하는 authenticating proxy을 사용할 때 유용하다.
- --requestheader-username-headers Required, case-insensitive. Header names to check, in order, for the user identity. The first header containing a value is used as the username.
- --requestheader-group-headers 1.6+. Optional, case-insensitive. "X-Remote-Group" is suggested. Header names to check, in order, for the user's groups. All values in all specified headers are used as group names.
- --requestheader-extra-headers-prefix 1.6+. Optional, case-insensitive. "X-Remote-Extra-" is suggested. Header prefixes to look for to determine extra information about the user (typically used by the configured authorization plugin). Any headers beginning with any of the specified prefixes have the prefix removed. The remainder of the header name is lowercased and percent-decoded and becomes the extra key, and the header value is the extra value.

## Anonymous requests
해당 기능이 활성화되면, 다른 authentication에 의해 거절되지 않은 요청은 anonymous request로 간주되며 사용자 이름 `system:anonymous`, 그룹 `system:unauthenticated`에 속하게 된다.

예를 들어, token authentication이 설정된 서버에서 anonymous access가 활성화된 경우, 잘못된 bearer token을 사용하는 요청은 401 Unauthorized 오류를 응답받게 된다. 하지만 bearere token을 사용하지 않는 요청은 anonymous request로 처리된다.

1.5.1-1.5.x 버전에서는 anonymous access가 기본적으로 비활성화되어 있으며, kube-apiserver에 `--anonymous-auth=true` flag를 사용해 활성화할 수 있다.

1.6 이상에서는 AlwaysAllow 이외의 authorization 모드를 사용하는 경우 기본적으로 anonymous access가 활성화되며, kube-apiserver에 `--anonymous-auth=false`를 사용해 비활성화할 수 있다. 1.6 버전부터 ABAC, RBAC authorizer는 `system:anonymous` 사용자 또는 `system:unauthenticated` 그룹에 대한 명시적 권한 부여를 요구하기 때문에 `*` 사용자, `*` 그룹에 접근 권한을 부여하는 기존 legacy 정책은 anonymous 사용자를 포함하지 않는다.

## User impersonation
사용자는 impersonation header를 사용해 다른 사용자처럼 행동할 수 있다. 이를 통해 요청이 인증된 사용자 정보를 무시할 수 있다. 예를 들어, 관리자는 이 기능을 사용해 일시적으로 다른 사용자로 위장하고 요청이 거부되는지 확인해 권한 정책을 디버깅할 수 있다.

impersonation request은 먼저 요청 사용자로 인증한 다음 impersonated user 사용자로 스위칭하게 된다.
- 사용자가 자신의 credential과 impersonation header를 사용하여 API를 호출한다.
- kube-apiserver는 사용자를 인증한다.
- kube-apiserver는 인증된 사용자가 위장 권한(privilege)을 가지고 있는지 확인한다.
- 요청 사용자 정보가 위장 값으로 대체된다.
- 요청이 평가되며 인가는 위장된 사용자 정보에 적용된다.

다음 HTTP header를 사용하여 위장 요청을 수행할 수 있다.
- `Impersonate-User`: 위장할 사용자 이름
- `Impersonate-Group`: (optional)위장할 그룹 이름. 여러 그룹을 명시하기 위해 여러번 header를 사용할 수 있다. `Impersonate-User`가 필요하다.
- `Impersonate-Extra-( extra name )`: (optional)사용자와 추가 필드를 연결하는 dynamic header다. `Impersonate-User`가 필요. 일관되게 보존하기 위해 ( extra name )은 소문자를 사용하고 [legal in HTTP header labels](https://datatracker.ietf.org/doc/html/rfc7230#section-3.2.6)가 아닌 문자는 utf-8, [percent-encoded](https://datatracker.ietf.org/doc/html/rfc3986#section-2.1)이어야 한다.
- `Impersonate-Uid`: (optional)위장된 사용자를 나타내는 uid다. `Impersonate-User`가 필요합니다. k8s는 이 문자열에 대해 제약 사항을 갖지 않는다.

> **Note**:  
> Prior to 1.11.3 (and 1.10.7, 1.9.11), ( extra name ) could only contain characters which were [legal in HTTP header labels](https://datatracker.ietf.org/doc/html/rfc7230#section-3.2.6).

> **Note**:  
> `Impersonate-Uid`은 1.22 버전 이상부터 유효하다.

아래는 특정 사용자, 그룹으로 위장하기 위한 HTTP 요청 header 예시다.
```
Impersonate-User: jane.doe@example.com
Impersonate-Group: developers
Impersonate-Group: admins
```

아래는 추가 기타 필드, uid를 설정하는 HTTP 요청 header 예시다.
```
Impersonate-User: jane.doe@example.com
Impersonate-Extra-dn: cn=jane,ou=engineers,dc=example,dc=com
Impersonate-Extra-acme.com%2Fproject: some-project
Impersonate-Extra-scopes: view
Impersonate-Extra-scopes: development
Impersonate-Uid: 06f6ce97-e2c5-4ab8-7ba5-7654dd08d52b
```

kubectl에서 `Impersonate-User` header를 설정하기 위해 `--as` flag, `Impersonate-Group` header를 설장하기 위해 `--as-group` flag를 사용할 수 있다.
``` sh
kubectl drain mynode
Error from server (Forbidden): User "clark" cannot get nodes at the cluster scope. (get nodes mynode)

kubectl drain mynode --as=superman --as-group=system:masters
node/mynode cordoned
node/mynode drained
```

> **Note**:  
> kubectl은 기타 필드, uid를 설정할 수 없다.

사용자, 그룹, uid, 추가 필드를 위장하기 위해 사용자는 위장 대상 속성("user", "group", "uid" 등)에 대해 "impersonate" 동사를 수행할 수 있어야 . RBAC authorization 플러그인을 사용하는 경우, 다음 ClusterRole은 사용자, 그룹 위장 header를 설정하는 데 필요한 규칙을 포함한다.
``` yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: impersonator
rules:
- apiGroups: [""]
  resources: ["users", "groups", "serviceaccounts"]
  verbs: ["impersonate"]
```

기타 필드, uid는 "authentication.k8s.io" apiGroups에 속한다. 기타 필드는 "userextras" resource의 sub-resource로 평가된다. 사용자가 기타 필드 "scopes", uid를 위장하기 위해 필요한 규칙은 다음과 같다.
``` yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: scopes-and-uid-impersonator
rules:
# Can set "Impersonate-Extra-scopes" header and the "Impersonate-Uid" header.
- apiGroups: ["authentication.k8s.io"]
  resources: ["userextras/scopes", "uids"]
  verbs: ["impersonate"]
```

위장 헤더의 값은 `resourceNames`를 사용해 제한할 수 있다.
``` yaml
kind: ClusterRole
metadata:
  name: limited-impersonator
rules:
# Can impersonate the user "jane.doe@example.com"
- apiGroups: [""]
  resources: ["users"]
  verbs: ["impersonate"]
  resourceNames: ["jane.doe@example.com"]

# Can impersonate the groups "developers" and "admins"
- apiGroups: [""]
  resources: ["groups"]
  verbs: ["impersonate"]
  resourceNames: ["developers","admins"]

# Can impersonate the extras field "scopes" with the values "view" and "development"
- apiGroups: ["authentication.k8s.io"]
  resources: ["userextras/scopes"]
  verbs: ["impersonate"]
  resourceNames: ["view", "development"]

# Can impersonate the uid "06f6ce97-e2c5-4ab8-7ba5-7654dd08d52b"
- apiGroups: ["authentication.k8s.io"]
  resources: ["uids"]
  verbs: ["impersonate"]
  resourceNames: ["06f6ce97-e2c5-4ab8-7ba5-7654dd08d52b"]
```

> **Note**:  
> 사용자 또는 그룹을 위장하면 해당 사용자 또는 그룹이 되는 것처럼 모든 작업을 수행할 수 있다. 이러한 이유로 위장은 ns에 제한되지 않는다. k8s RBAC를 사용하여 위장을 허용하려면 Role, RoleBinding이 아니라 ClusterRole, ClusterRoleBinding을 사용해야 한다.

## client-go credential plugins
`k8s.io/client-go`를 사용하면 외부 외부 명령어를 실행해 사용자 credential을 얻을 수 있다(예를 들어 kubectl, kubelet이 사용).
 
이러한 기능은 `k8s.io/client-go`에서 내장 지원하지 않는 인증 프로토콜(LDAP, Kerberos, OAuth2, SAML 등)과의 통합이 목적이다 플러그인은 각 프로토콜의 로직을 구현하며 결과적으로 opaque credential을 반환한다. Almost all credential plugin use cases require a server side component with support for the webhook token authenticator to interpret the credential format produced by the client plugin.

> **Note**:
> 이전에 kubectl은 AKS, GKE에 대한 인증을 내장 지원했으나 지금은 그렇지 않다.

### Example use case
예를 들어, 어떤 조직이 LDAP 사용자 credential을 signed token으로 교환하는 외부 서비스를 운영한다. 이 서비스는 token을 검증하기 위해 [webhook token authenticator](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#webhook-token-authentication) 역할도 수행할 수 있다(즉, kube-apiserver의 `TokenReview` 요청을 처리할 수 있다). 이 경우 사용자는 로컬 환경에 credential plugin을 설치해야 한다.

인증을 위한 절차는 다음과 같다.
1. 사용자는 `kubectl` 명령을 실행한다.
2. `kubectl`이 사용하는 client-go는 credential plugin에게 `ExecCredential` object을 입력 변수로 사용해 외부 명령어 실행을 요청한다.
2. (필요할 경우) credential plugin은 프롬프트를 통해 사용자의 LDAP credential 입력을 요청하고, 입력된 credential을 사용해 외부 서비스에 signed token을 얻는다.
3. credential plugin은 `ExecCredential` object의 `.status` 필드를 사용해 `client-go`에 signed token을 반환하며 `client-go`는 signed token을 bearer token으로 사용해 kube-apiserver에 요청을 수행한다.
4. kube-apiserver는 webhook token authenticator로 등록된 외부 서비스에 `TokenReview` API를 요청한다.
5. 외부 서비스는 token의 서명을 검증하고, 결과로 사용자 이름과 그룹을 반환한다.

### Configuration
[kubectl config file](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/)에서 credential plugin은 `.users[*].user.exec` 필드를 통해 구현된다(`.users[*].name.exec.apiVersion` 필드는 client-go가 credential plugin 호출 시, 사용할 ExecCredential object의 api version에 대한 설정이다).
- client.authentication.k8s.io/v1
  ``` yaml
  apiVersion: v1
  kind: Config
  users:
  - name: my-user
    user:
      exec:
        # Command to execute. Required.
        command: "example-client-go-exec-plugin"

        # API version to use when decoding the ExecCredentials resource. Required.
        #
        # The API version returned by the plugin MUST match the version listed here.
        #
        # To integrate with tools that support multiple versions (such as client.authentication.k8s.io/v1beta1),
        # set an environment variable, pass an argument to the tool that indicates which version the exec plugin expects,
        # or read the version from the ExecCredential object in the KUBERNETES_EXEC_INFO environment variable.
        apiVersion: "client.authentication.k8s.io/v1"

        # Environment variables to set when executing the plugin. Optional.
        env:
        - name: "FOO"
          value: "bar"

        # Arguments to pass when executing the plugin. Optional.
        args:
        - "arg1"
        - "arg2"

        # Text shown to the user when the executable doesn't seem to be present. Optional.
        installHint: |
          example-client-go-exec-plugin is required to authenticate
          to the current cluster.  It can be installed:

          On macOS: brew install example-client-go-exec-plugin

          On Ubuntu: apt-get install example-client-go-exec-plugin

          On Fedora: dnf install example-client-go-exec-plugin

          ...        

        # Whether or not to provide cluster information, which could potentially contain
        # very large CA data, to this exec plugin as a part of the KUBERNETES_EXEC_INFO
        # environment variable.
        provideClusterInfo: true

        # The contract between the exec plugin and the standard input I/O stream. If the
        # contract cannot be satisfied, this plugin will not be run and an error will be
        # returned. Valid values are "Never" (this exec plugin never uses standard input),
        # "IfAvailable" (this exec plugin wants to use standard input if it is available),
        # or "Always" (this exec plugin requires standard input to function). Required.
        interactiveMode: Never
  clusters:
  - name: my-cluster
    cluster:
      server: "https://172.17.4.100:6443"
      certificate-authority: "/etc/kubernetes/ca.pem"
      extensions:
      - name: client.authentication.k8s.io/exec # reserved extension name for per cluster exec config
        extension:
          arbitrary: config
          this: can be provided via the KUBERNETES_EXEC_INFO environment variable upon setting provideClusterInfo
          you: ["can", "put", "anything", "here"]
  contexts:
  - name: my-cluster
    context:
      cluster: my-cluster
      user: my-user
  current-context: my-cluster
  ```
- client.authentication.k8s.io/v1beta1
  ``` yaml
  apiVersion: v1
  kind: Config
  users:
  - name: my-user
    user:
      exec:
        # Command to execute. Required.
        command: "example-client-go-exec-plugin"

        # API version to use when decoding the ExecCredentials resource. Required.
        #
        # The API version returned by the plugin MUST match the version listed here.
        #
        # To integrate with tools that support multiple versions (such as client.authentication.k8s.io/v1),
        # set an environment variable, pass an argument to the tool that indicates which version the exec plugin expects,
        # or read the version from the ExecCredential object in the KUBERNETES_EXEC_INFO environment variable.
        apiVersion: "client.authentication.k8s.io/v1beta1"

        # Environment variables to set when executing the plugin. Optional.
        env:
        - name: "FOO"
          value: "bar"

        # Arguments to pass when executing the plugin. Optional.
        args:
        - "arg1"
        - "arg2"

        # Text shown to the user when the executable doesn't seem to be present. Optional.
        installHint: |
          example-client-go-exec-plugin is required to authenticate
          to the current cluster.  It can be installed:

          On macOS: brew install example-client-go-exec-plugin

          On Ubuntu: apt-get install example-client-go-exec-plugin

          On Fedora: dnf install example-client-go-exec-plugin

          ...        

        # Whether or not to provide cluster information, which could potentially contain
        # very large CA data, to this exec plugin as a part of the KUBERNETES_EXEC_INFO
        # environment variable.
        provideClusterInfo: true

        # The contract between the exec plugin and the standard input I/O stream. If the
        # contract cannot be satisfied, this plugin will not be run and an error will be
        # returned. Valid values are "Never" (this exec plugin never uses standard input),
        # "IfAvailable" (this exec plugin wants to use standard input if it is available),
        # or "Always" (this exec plugin requires standard input to function). Optional.
        # Defaults to "IfAvailable".
        interactiveMode: Never
  clusters:
  - name: my-cluster
    cluster:
      server: "https://172.17.4.100:6443"
      certificate-authority: "/etc/kubernetes/ca.pem"
      extensions:
      - name: client.authentication.k8s.io/exec # reserved extension name for per cluster exec config
        extension:
          arbitrary: config
          this: can be provided via the KUBERNETES_EXEC_INFO environment variable upon setting provideClusterInfo
          you: ["can", "put", "anything", "here"]
  contexts:
  - name: my-cluster
    context:
      cluster: my-cluster
      user: my-user
  current-context: my-cluster
  ```

상대 경로로 표현된 명령어는 kubeconfig 파일의 디렉토리 기준으로 계산된다. If KUBECONFIG is set to /home/jane/kubeconfig and the exec command is ./bin/example-client-go-exec-plugin, the binary /home/jane/bin/example-client-go-exec-plugin is executed.
``` yaml
- name: my-user
  user:
    exec:
      # Path relative to the directory of the kubeconfig
      command: "./bin/example-client-go-exec-plugin"
      apiVersion: "client.authentication.k8s.io/v1"
      interactiveMode: Never
```

아래는 aws eks에 대한 예시다.
1. kubectl 명령어 사용을 위한 kubeconfig 파일 업데이트
    ``` sh
    aws eks update-kubeconfig --name ${EKS_CLUSTER} --alias ${EKS_CLUSTER} --profile ${AWS_PROFILE} --user-alias ${EKS_CLUSTER}
    ```
2. 아래는 kubeconfig 파일의 일부 내용이다.
    ``` yaml
    (...생략...)
    users:
      - name: ${EKS_CLUSTER}
        user:
          exec:
            apiVersion: client.authentication.k8s.io/v1beta1
            args:
              - --region
              - ap-northeast-2
              - eks
              - get-token
              - --cluster-name
              - ${EKS_CLUSTER}
              - --output
              - json
            command: aws
            env:
              - name: AWS_PROFILE
                value: ${AWS_PROFILE}
            interactiveMode: IfAvailable
            provideClusterInfo: false
    (...생략...)
    ```
3. kubectl 명령어는 kubeconfig 파일에 설정된 client-go credential plugin 정보를 사용해 외부 명령어(aws eks get-token)를 실행하고 token을 얻는다.
4. kubectl은 해당 token을 사용해 kube-apiserver에 요청을 보낸다.
5. kube-apiserver은 aws-iam-authenticator(webhook token authenticator)에 TokenReview API를 요청하고 사용자 이름, 그룹에 대한 정보를 반환받는다.
6. 이후 kube-apiserver는 authorization, admission controller 단계를 거쳐 요청에 대한 응답을 수행한다.

아래는 eks cluster에 대해 token을 직접 발급받고 요청을 수행하는 예시다.
``` sh
# kubectl
kubectl get ns --context ${EKS_CLUSTER} --token=$(aws eks get-token --cluster-name ${EKS_CLUSTER} --profile ${AWS_PROFILE} --query 'status.token' --output text)

# curl
curl -H "Authorization: Bearer $(aws eks get-token --cluster-name ${EKS_CLUSTER} --profile ${AWS_PROFILE} --query 'status.token' --output text)" -k ${EKS_APISERVER_ENDPOINT}/api/v1/namespaces
```

### Input and output formats
외부 명령어는 ExecCredential object를 stdout에 출력한다. `k8s.io/client-go`는 반환된 credential의 `.status` 필드를 참고해 kube-aoiserver에 인증한다. 실행된 외부 명령어는 `KUBERNETES_EXEC_INFO` 환경 변수를 통해 ExecCredential object를 입력 값으로 받는다. 이 입력에는 반환될 ExecCredential object의 예상 API 버전, 플러그인이 사용자와 상호 작용하기 위해 stdin을 사용할 수 있는지 여부와 같은 유용한 정보가 포함되어 있다.

대화형 세션(즉, 터미널)에서 실행될 때 stdin은 플러그인에 직접 노출될 수 있다. 플러그인은 `KUBERNETES_EXEC_INFO` 환경 변수에서 가져온 입력 ExecCredential object의 `spec.interactive` 필드를 사용해 stdin이 제공되었는지 확인해야 한다. 플러그인의 stdin 요구 사항(optional, required, never)은 kubeconfig의 `user.exec.interactiveMode` 필드를 통해 설정된니다. `user.exec.interactiveMode` 필드는 `client.authentication.k8s.io/v1beta1`에서는 선택 사항이고 `client.authentication.k8s.io/v1`에서는 필수다.

## API access to authentication information for a client
cluster에 API가 활성화되어 있다면 `SelfSubjectReview` API를 사용하여 k8s cluster가 인증 정보를 클라이언트로 식별하는 방법을 확인할 수 있다. 이 방법은 사용자(일반적으로 실제 사람을 나타냄) 또는 sa로 인증하는 경우에 모두 동작한다.

`SelfSubjectReview` object에는 설정 가능한 필드가 없다. 요청을 받으면 kube-apiserver가 사용자 속성을 status에 채워 응답한다.

아래는 요청 예시다.
```
POST /apis/authentication.k8s.io/v1/selfsubjectreviews

{
  "apiVersion": "authentication.k8s.io/v1",
  "kind": "SelfSubjectReview"
}
```

아래는 응답 예시다.
``` json
{
  "apiVersion": "authentication.k8s.io/v1",
  "kind": "SelfSubjectReview",
  "status": {
    "userInfo": {
      "name": "jane.doe",
      "uid": "b6c7cfd4-f166-11ec-8ea0-0242ac120002",
      "groups": [
        "viewers",
        "editors",
        "system:authenticated"
      ],
      "extra": {
        "provider_id": ["token.company.example"]
      }
    }
  }
}
```

`kubectl auth whoami` 명령어를 사용할 수 있다. 아래는 출력 예시다.
- 간단한 출력 예시
  ``` sh
  ATTRIBUTE         VALUE
  Username          jane.doe
  Groups            [system:authenticated]
  ```
- 추가 필드를 포함하는 출력 예싣
  ``` sh
  ATTRIBUTE         VALUE
  Username          jane.doe
  UID               b79dbf30-0c6a-11ed-861d-0242ac120002
  Groups            [students teachers system:authenticated]
  Extra: skills     [reading learning]
  Extra: subjects   [math sports]
  ```

kubectl의 `--output` flag를 사용해 JSON, YAML 포맷의 데이터를 확인할 수도 있다.
``` json
  "apiVersion": "authentication.k8s.io/v1",
  "kind": "SelfSubjectReview",
  "status": {
    "userInfo": {
      "username": "jane.doe",
      "uid": "b79dbf30-0c6a-11ed-861d-0242ac120002",
      "groups": [
        "students",
        "teachers",
        "system:authenticated"
      ],
      "extra": {
        "skills": [
          "reading",
          "learning"
        ],
        "subjects": [
          "math",
          "sports"
        ]
      }
    }
  }
}
```

``` yaml
apiVersion: authentication.k8s.io/v1
kind: SelfSubjectReview
status:
  userInfo:
    username: jane.doe
    uid: b79dbf30-0c6a-11ed-861d-0242ac120002
    groups:
    - students
    - teachers
    - system:authenticated
    extra:
      skills:
      - reading
      - learning
      subjects:
      - math
      - sports
```

k8s에 webhook token authentication, authenticating proxy와 같은 복잡한 authentication을 사용하는 경우 유용하다.

> **Note**:  
> kube-apiserver는 위장을 포함한 모든 authentication을 적용한 후 userInfo를 채운다. 따라서 위장을 사용하는 경우 `SelfSubjectReview` API 요청은 위장된 사용자의 세부 정보를 확인할 수 있다.

기본적으로 `APISelfSubjectReview` feature가 활성화된 경우 모든 인증된 사용자는 `SelfSubjectReview` object를 생성할 수 있다. 이는 `system:basic-user` cluster role에 의해 허용된다.

> **Note**:  
> 아래 경우에만 `SelfSubjectReview` 요청을 수행할 수 있다.
> - APISelfSubjectReview feature gate가 활성화된 경우(k8s 1.30에서는 활성화 할 필요가 없다)
> - (if you are running a version of Kubernetes older than v1.28) the API server for your cluster has the authentication.k8s.io/v1alpha1 or authentication.k8s.io/v1beta1 API group enabled.