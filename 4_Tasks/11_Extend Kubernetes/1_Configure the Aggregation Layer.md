aggregation layer를 구성하면 core k8s API의 일부가 아닌 추가 API를 통해 k8s API server를 확장할 수 있다.

## Before you begin
**Note**: proxy와 extension apiserver간 상호 TLS 인증을 지원하는 환경에서 aggregation layer를 활성화하기 위해 몇 가지 설정 요구 사항이 있다. k8s와 kube-apiserver에는 여러 CA가 있으므로 proxy가 k8s 일반 CA 같은 것이 아니라 aggregation layer CA에 의해 서명되었는지 확인해야 한다.

**Caution**: 다른 클라이언트 타입을 위한 CA를 재사용하는 것은 클러스터 기능에 부정적인 영향을 미칠 수도 있다.

## Authentication Flow
crd와 다르게 aggregation API는 k8s 표준 apiserver에 추가적으로 extension apiserver라는 다른 서버를 포함한다. k8s apiserver는 extension apiserver와 통신할 수 있어야하며 반대도 마찬가지다. 이러한 통신에 대한 보안을 위해 k8s apiserver는 X509 인증서를 사용해 extension apiserver에 대해 인증한다.

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

#### Original Request Username and Group

### Extension Apiserver Authenticates the Request

### Extension Apiserver Authorizes the Request

### Extension Apiserver Executes

## Enable Kubernetes Apiserver flags