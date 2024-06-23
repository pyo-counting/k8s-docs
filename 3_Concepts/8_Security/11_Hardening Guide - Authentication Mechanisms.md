
적절한 인증 메커니즘을 선택하는 것은 cluster 보안에 중요하다. k8s는 강점과 약점을 가진 여러 메커니즘을 내장 제공하한다. 관리자는 cluster에 가장 적합한 인증 메커니즘을 선택하는 것이 중요하다.

일반적으로는 사용자 관리를 단순화, 더 이상 유효하지 않은 사용자의 접근을 방지하기 위해 적은 수의 인증 메커니즘을 활성화하는 것이 좋다.

k8s는 cluster 내에 내장된 사용자 데이터베이스를 가지고 있지 않다. 대신 관리자가 설정한 인증 시스템에서 사용자 정보를 받아 인가에 대한 결정을 내린다. 따라서 사용자 접근을 감사하기 위해서는 관리자가 설정한 모든 인증 시스템에서 제공되는 credential을 검토해야 한다.

여러 사용자가 k8s API에 직접 접근하는 운영 환경 cluster의 경우 OIDC와 같은 외부 인증 시스템을 사용하는 것이 좋다. client 인증서와 sa token과 같은 k8s 내장 인증 메커니즘은 운영 환경에서 적합하지 않다.

## X.509 client certificate authentication
kubelet이 kube-apiserver에 인증할 때와 같이 k8s 구성 요소를 위해 [X.509 client certificate](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#x509-client-certificates) 인증을 사용한다. 이 메커니즘은 사용자 인증에도 사용할 수 있지만 몇 가지 제한 사항이 있기 때문에 운영 환경에서는 적합하지 않을 수 있다.
- 클라이언트 인증서는 개별적으로 무효화할 수 없다. 만약 유출된다면 만료될 때까지 공격자에 의해 사용될 수도 있다. 이 위험을 완화하기 위해, 클라이언트 인증서를 사용해 생성된 사용자 authentication credential에 짧은 수명을 설정하는 것을 권장한다.
- 인증서를 무효화하기 위해서 ca(Certificate Authority)을 re-key해야하며 이는 cluster의 가용성에 위험을 초래할 수 있다.
- cluster 내에 생성된 클라이언트 인증서의 영구적으로 기록되지 않는다. 그렇기 때문에 발급된 모든 인증서는 추적이 필요할 경우 직접 기록해야 한다.
- 클라이언트 인증서 인증에 사용되는 private key는 암호화할 수 없다. 키가 포함된 파일을 읽을 수 있는 사람은 누구든지 이를 사용할 수 있다.
- 클라이언트 인증서 인증을 사용하기 위해 중간 TLS 종료 지점 없이 클라이언트가 kube-apiserver에 직접 연결해야 하며 이는 네트워크 아키텍처를 복잡하게 만들 수 있다.
- 그룹 데이터는 클라이언트 인증서의 O 값에 포함되어 있으므로 인증서의 유효 기간 동안 사용자가 속한 그룹을 변경할 수 없다.

## Static token file
k8s는 control plane을 실행 중인 호스트의 디스크에 저장된 [static token file](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#static-token-file)에서 credential을 로드할 수 있지만 이 접근 방식은 여러 가지 이유로 운영 환경에 적합하지 않다.
- credential이 control plane을 실행 중인 디스크에 평문으로 저장되므로 보안 위험이 있을 수 있다.
- credential을 변경하려면 kube-apiserver 프로세스를 재시작해야 하므로 가용성에 영향을 미칠 수 있다.
- 사용자가 credential을 변경할 수 있는 메커니즘이 없다. credential을 변경하기 위해 cluster 관리자는 직접 디스크에 저장된 파일에서 토큰 값을 수정해야 한다.
- brute-force 공격을 방지할 수 있는 lockout 메커니즘이 없다.

## Bootstrap tokens
[bootstrap token](https://kubernetes.io/docs/reference/access-authn-authz/bootstrap-tokens/)은 no가 k8s cluster에 가입(join)하기 위해 사용되며 여러 이유로 사용자 인증에 권장되지 않는다.
- bootstrap token의 그룹은 하드코딩되어 있기 때문에 일반 사용자를 위해서는 부적합하다.
- 직접 생성한 bootstrap token은 보안이 취약할 수 있으며 이는 공격자가 추측할 수 있어 보안 위험이 된다.
- brute-force 공격을 방지할 수 있는 lockout 메커니즘이 없다.

## ServiceAccount secret tokens
[service account secrets](https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/#manual-secret-management-for-serviceaccounts)을 사용해 cluster에서 실행 중인 workload가 kube-apiserver에 인증할 수 있다. k8s 1.23 이하 버전에서는 이 옵션이 기본 값이었으나, 이제 TokenRequest API token으로 대체됐다. sa secret을 사용자 인증에 사용할 수 있지만 여러가지 이유로 부적합하다.
- token의 만료기간을 설정할 수없으며 연결된 sa가 삭졔될 때까지 유효하다.
- token이 저장된 ns에서 secret에 대한 read 권한이 있는 모든 사용자가 조회할 수 있다.
- sa을 임의의 group에 추가할 수 없기 때문에 RBAC에 대한 관리가 복잡해질 수 있다.

## TokenRequest API tokens
TokenRequest API는 kube-apiserver 또는 third-party 시스템에 인증을 위해 단기 credential을 생성하는 데 유용하다. 그러나 무효화 방법이 없고 credential을 안전하게 사용자에게 배포하는 것이 어려워 사용자 인증에는 일반적으로 권장되지 않는다.

TokenRequest 토큰을 사용할 때는 유출된 토큰으로 인한 피해를 줄이기 위해 짧은 만료 기간을 설정하는 것을 권장한다.

## OpenID Connect token authentication
k8s는 [OpenID Connect (OIDC)](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#openid-connect-tokens)를 사용해 k8s API와 외부 인증 서비스를 통합하는 것을 지원한다. k8s를 IdP(identity provider)와 통합하기 위해 사용할 수 있는 다양한 소프트웨어가 있다. 그러나 k8s의 OIDC 인증을 사용할 때는 강력한 보완을 위한 아래 내용을 고려해야 한다.
- cluster에 설치된 OIDC 인증을 지원하는 소프트웨어는 높은 권한으로 실행되므로 일반 workload와 분리되어야 한다.
- 일부 k8s 관리 서비스는 사용할 수 있는 OIDC provier가 제한된다.
- TokenRequest API token과 마찬가지로 OIDC token도 유출된 토큰으로 인한 피해를 줄이기 위해 짧은 만료 기간을 설정하는 것을 권장한다.

## Webhook token authentication
[webhook token authentication](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#webhook-token-authentication)은 외부 인증 provider를 k8s에 통합하는 또 다른 옵션이다. 이 메커니즘은 cluster 내 또는 외부에서 실행되는 인증 서비스가 웹훅을 통해 인증 결정을 내리도록 허용한다. 이 메커니즘의 적합성은 인증 서비스에 사용되는 소프트웨어에 따라 달라질 수 있으며 k8s를 위한 고려 사항이 있다.

웹훅 인증을 설정하기 위해서는 control plane을 실행 중인 호스트의 파일 시스템에 접근이 필요하다(설정 파일을 위해). 이는 managed k8s에서 provider가 제공하지 않는 한 불가능하다. 그리고 cluster에 설치된 웹훅 인증 소프트웨어는 높은 권한으로 실행되므로 일반 workload와 분리되어야 한다.

## Authenticating proxy
k8s에 외부 인증 시스템을 통합하는 또 다른 옵션은 [authenticating proxy](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#authenticating-proxy)를 사용하는 것이다. 이 메커니즘을 사용하면 k8s는 특정 HTTP header가 설정된 proxy로부터 요청을 받아 사용자와 그룹을 매핑한다. 이 메커니즘을 사용할 때는 몇 가지 고려 사항이 있다.

먼저, proxy와 k8s kube-apiserver 간의 TLS가 안전하게 구성되어야 traffic interception, sniffing attack 위험을 줄일 수 있다. 이는 proxy와 k8s kube-apiserver 간의 통신이 안전하도록 보장한다.

둘째, 요청 HTTP header를 수정할 수 있는 공격자가 k8s resource에 무단 접근할 수 있으므로 HTTP header가 제대로 보안되고 변조되지 않도록 하는 것이 중요하다.