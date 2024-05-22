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

### Webhook Token Authentication
### Authenticating Proxy
## Anonymous requests
## User impersonation
## client-go credential plugins
### Example use case
### Configuration
### Input and output formats
## API access to authentication information for a client