사용자와 k8s sa에 API 접근 권한을 부여할 수 있다. When a request reaches the API, it goes through several stages, illustrated in the following diagram:

![API server authentication, authorization, admission controll](https://d33wubrfki0l68.cloudfront.net/673dbafd771491a080c02c6de3fdd41b09623c90/50100/images/docs/admin/access-control-overview.svg)

## Transport security
기본적으로 kube-apiserver는 첫 번째 non-localhost 네트워크 인터페이스의 6443번 포트에서 listen(TLS 사용)한다. 운영 k8s 환경에서는 일반적으로 443번 포트에서 서비스를 제공한다. 포트는 `--secure-port`, IP는 `--bind-address` flag로 변경할 수 있다.

kube-apiserver는 인증서를 제공한다. 이 인증서는 사설 CA 또는 일반적인 PKI(publick key infrastructure) 기반 CA에 의해 서명된다. 인증서와 private key는 `--tls-cert-file`, `--tls-private-key-file` flag로 변경할 수 있다.

사설 CA를 사용하는 경우 client의 ~/.kube/config에 해당 CA 인증서 사본을 설정해야 한다.

이 단계에서 클라이언트는 자신의 TLS 인증서를 제시할 수 있다.

## Authentication
TLS 연결이 수립(established)되면 HTTP 요청은 authentication 단계로 이동한다. 이는 위 그림에서 step 1이다. kube-apiserver는 1개 이상의 authenticator 모듈을 실행하도록 설정된다.

authentication를 위한 입력은 전체 HTTP 요청이지만 일반적으로 header, client 인증서(위 Transport security 단계에서 client가 보낸)을 검사한다.

authentication 모듈은 client certificate, password, plain token, bootstrap token, sa에서 사용하는 JWT(JSON Web Token)등이 있다.

여러 authentication 모듈을 사용할 수 있으며 이 경우 각 모듈은 인증이 성공할 떄까지 순서대로 실행된다.

요청에 대한 인증이 실패하면 HTTP status 401 code를 반환한다. 인증에 성공하면 사용자는 특정 `username`으로 인증되며 해당 이름은 이 후 authorization 단계에서 사용된다. 일부 authenticator는 사용자의 `group`도 제공한다.

k8s는 요청 로깅, 접근 제어에 대한 결정에 username을 사용하지만 별도로 user object가 없으며 API에 username이나 다른 정보를 저장하지 않는다.

## Authorization
HTTP 요청이 특정 사용자로부터 인입된 것으로 인증된 후에, 해당 요청은 인가(authorization)돼야 한다. 이는 위 그림에서 step 2이다.

요청에는 요청자의 이름(username), 행동(action), 행동의 영향을 받는 object가 포함되어야 한다. kube-apiserver에 정의된 정책에 의해 사용자, 행동에 대한 권한이 있다면 요청은 인가된다.

예를 들어 Bob이 아래와 같은 정책을 갖는다면 projectCaribou ns의 po에 대해 read 권한이 있다.
``` json
{
    "apiVersion": "abac.authorization.kubernetes.io/v1beta1",
    "kind": "Policy",
    "spec": {
        "user": "bob",
        "namespace": "projectCaribou",
        "resource": "pods",
        "readonly": true
    }
}
```

Bob의 아래 요청은 projectCaribou ns에 대해 object read 권한이 있기 때문에 요청이 인가된다.
``` json
{
  "apiVersion": "authorization.k8s.io/v1beta1",
  "kind": "SubjectAccessReview",
  "spec": {
    "resourceAttributes": {
      "namespace": "projectCaribou",
      "verb": "get",
      "group": "unicorn.example.org",
      "resource": "pods"
    }
  }
}
```

Bob이 projectCaribou ns의 object에 대해 write(create, update)를 요청하면 거절된다. 그리고 다른 ns의 object에 대해 read를 요청하면 거절된다.

k8s authorization은 기존의 조직 또는 cloud provier 접근 제어 시스템과 상호 작용하기 위해 공통 REST attribute를 사용해야 한다. 이러한 제어 시스템은 k8s API 외에 다른 API와 상호 작용할 수 있으므로 REST 포맷을 사용하는 것이 중요하다.

k8s는 ABAC, RBAC, WebHook과 같은 여러 authorization 모듈을 제공한다. 관리자는 k8s cluster 생성 시 여러 authorization 모듈을 설정할 수 있다. 2개 이상의 모듈이 설정된 경우 1개의 모듈에서 인가가 완료되면 다른 모듈은 실행하지 않는다. 모든 모듈에서도 인가되지 않는다면 HTTP status 403 코드를 반환한다.

## Admission control
admission control 모듈은 요청을 수정/거부할 수 있다. authorization 모듈에 사용 가능한 속성 외에도 admission control 모듈은 생성/수정 중인 object의 내용에 접근할 수 있다.

admission controller는 object를 생성, 수정, 삭제, 연결(proxy)하는 요청에 대해 동작한다. admission controller는 단순히 object를 읽는 요청에 대해서만 동작하지 않는다. 여러 admission control 모듈이 설정되면 순서대로 호출된다.

이는 위 그림에서 step 3이다.

authentication, authorization 모듈과 다르게 admission control 모듈 1개가 거절을 하면 해당 요청은 거절된다.

object를 거절하는 것 외에도 admission control 모듈은 object 필드에 대해 복잡한 기본값을 설정할 수도 있다.

요청이 모든 admission control 모듈을 통과하면 해당 API object에 대한 유효성 검사 루틴을 사용해 유효성이 검사되고 object store에 쓰여진다(위 그림에서 step 4).

## Auditing
k8s auditing은 cluster의 action 순서를 문서화하는 보안 관련 기록을 제공한다. cluster는 사용자, user, k8s API를 사용하는 애플리케이션, control plane 내 작업을 audit한다.