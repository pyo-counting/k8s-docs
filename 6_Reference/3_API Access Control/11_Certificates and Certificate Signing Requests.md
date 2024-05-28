k8s 인증서, trust bundle API는 클라이언트가 k8s API를 통해 X.509 인증서를 ca에 요청, 획득할 수 있는 프로그래밍 인터페이스를 제공하여 [X.509](https://www.itu.int/rec/T-REC-X.509) 자격 증명 프로비저닝을 자동화한다.

trust bundle에 대해서는 experimental (alpha) 기능으로 지원된다.

## Certificate signing requests
csr(CertificateSigningRequest) resource는 지정된 signer에 의해 인증서가 서명되도록 요청하기 위해 사용한다. 요청은 최종적으로 서명되기 전에 승인 또는 거부될 수 있다.

### Request signing process
csr resource를 통해 클라이언트는 서명 요청을 통해 X.509 인증서 발급을 요청할 수 있다. csr object는 `.spec.request` 필드에 PEM-encoded PKCS#10 서명 요청을 포함한다. csr은 `.spec.signerName` 필드에 signer를 지정한다. 해당 필드는 `certificates.k8s.io/v1` 버전 이후 필수 필드다. k8s v1.22 이상에서 클라이언트는 `.spec.expirationSeconds` 필드를 사용해 발급된 인증서의 수명을 지정할 수 있다. 이 필드의 최소 유효 값은 600, 즉 10분이다.

생성된 csr은 서명되기 전에 승인되어야 한다. 선택된 signer에 따라 csr은 kube-controller-manager에 의해 자동으로 승인될 수도 있다. 또는 `kubectl certificate approve`와 같이 수동으로 승인해야 한다. 거부도 할 수 있으며 이를 통해 signer에게 해당 요청이 서명되면 안된다는 것을 알린다.

인증서가 승인되면 서명 단계를 거친다. 관련된 signing controller는 먼저 서명 조건이 충족되었는지 확인한 후, 인증서를 생성한다. 그런 다음 signing controller는 새 인증서를 csr object의 `.status.certificate` 필드에 추가한다. `.status.certificate` 필드는 빈 값이거나 PEM-encoded X.509 인증서를 포함한다. csr의 `.status.certificate` 필드는 signer가 서명하기 전까지 빈 값이다.

`.status.certificate` 필드가 채워지면 csr 요청이 완료된 것이며 클라이언트는 csr object를 통해 서명된 인증서 PEM 데이터를 사용할 수 있다. 승인 조건이 충족되지 않았다면 signer는 인증서 서명을 거부할 수 있다.

cluster에 오래된 csr object의 개수를 줄이기 위해 정기적으로 gc controller가 동작한다. gc는 일정 기간 동안 상태가 변경되지 않은 csr를 삭제한다.
- 승인된 요청: 1시간 후 자동으로 삭제
- 거부된 요청: 1시간 후 자동으로 삭제
- 실패한 요청: 1시간 후 자동으로 삭제
- 대기 중인 요청: 24시간 후 자동으로 삭제
- 모든 요청: 발급된 인증서가 만료된 후 자동으로 삭제

### Certificate signing authorization
csr을 생성, 검색하기 위한 rbac는 다음과 같다:
- verbs: `create`, `get`, `list`, `watch`, apiGroups: `certificates.k8s.io`, resources: `certificatesigningrequests`

아래는 예시다.
``` yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: csr-creator
rules:
- apiGroups:
  - certificates.k8s.io
  resources:
  - certificatesigningrequests
  verbs:
  - create
  - get
  - list
  - watch
```

csr을 승인하기 위한 rbac는 다음과 같다.
- verbs: `get`, `list`, `watch`, apiGroups: `certificates.k8s.io`, resources: `certificatesigningrequests`
- verbs: `update`, apiGroups: `certificates.k8s.io`, resources: `certificatesigningrequests/approval`
- verbs: `approve`, apiGroups: `certificates.k8s.io`, resources: `signers`, resourceNames: `<signerNameDomain>/<signerNamePath> or <signerNameDomain>/*`

아래는 예시다.
``` yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: csr-signer
rules:
- apiGroups:
  - certificates.k8s.io
  resources:
  - certificatesigningrequests
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - certificates.k8s.io
  resources:
  - certificatesigningrequests/status
  verbs:
  - update
- apiGroups:
  - certificates.k8s.io
  resources:
  - signers
  resourceNames:
  - example.com/my-signer-name # example.com/* can be used to authorize for all signers in the 'example.com' domain
  verbs:
  - sign
```

## Signers
signer는 인증서를 서명을 수행하거나 서명을 수행할 수 있는 개체를 추상적으로 나타낸다.

cluster 외부에서 사용할 수 있는 signer에 대해서는 어떻게 동작하는지에 대한 정보를 제공해야 한다. 이를 통해 signer를 사용하는 입장에서 csr, ClusterTrustBundles에 대한 의미를 이해할 수 있다.

정보에는 아래 내용이 포함된다.
- Trust distribution: trust anchors(CA 인증서 또는 인증서 번들)가 어떻게 배포되는지
- Permitted subjects: 허용되지 않은 subjects가 요청될 때의 제한 사항 및 동작
- Permitted x509 extensions: IP subjectAltNames, DNS subjectAltNames, Email subjectAltNames, URI subjectAltNames 등을 포함하며, 허용되지 않은 extension이 요청될 때의 동작
- Permitted key usages / extended key usages: 요청 csr의 `.spec.usages` 값과 다른 사용 방법이 signer에 의해 지정된 경우의 제한 사항 및 동작
- Expiration/certificate lifetime: signer에 의해 고정 값인지 관리자에 의해 구성 가능한지, csr `.spec.expirationSeconds` 필드에 의해 결정되는지 등. 그리고 signer에 의해 결정된 만료 기간이 csr `.spec.expirationSeconds` 필드와 다를 때의 동작
- CA bit allowed/disallowed: signer가 허용하지 않을 때 csr이 ca 인증서를 요청하는 경우의 동작

일반적으로 csr의 `.status.certificate` 필드에는 승인 이후 발급된 단일 PEM-encoded X.509 인증서가 포함된다. 일부 signer는 여러 인증서를 `.status.certificate` 필드에 저장할 수 있다. 이 경우 signer에 대한 정보에 추가 인증서의 의미를 명시해야 한다. 예를 들어 이는 TLS handshake 중에 제시될 수 있는 인증서, 중간자일 수 있다.

If you want to make the trust anchor (root certificate) available, this should be done separately from a CertificateSigningRequest and its status.certificate field. For example, you could use a ClusterTrustBundle.

PKCS#10 서명 요청 형식은 인증서 만료 기간을 지정하는 표준 메커니즘을 갖고 있지 않다. 따라서 만료 기간은 csr의 `.spec.expirationSeconds` 필드를 통해 설정해야 한다. 내장 signer는 기본 값으로 kube-controller-manager의 `--cluster-signing-duration`(1년) 값을 사용한다. `.spec.expirationSeconds`가 지정된 경우에는 두 값 중 최소 값을 사용한다.

> **Note**:  
> `spec.expirationSeconds` 필드는 k8s v1.22에서 추가됐다. k8s의 이전 버전은 이 필드를 무시한다. v1.22 이전 버전의 kube-apiserver는 이 필드를 무시한다.

### Kubernetes signers
k8s는 아래와 같은 내장 `signerName`을 제공한다.
- `kubernetes.io/kube-apiserver-client`: signs certificates that will be honored as client certificates by the API server. Never auto-approved by kube-controller-manager.kube-controller-manager는 자동으로 승인을 수행하지 않는다.
  - Trust distribution: 서명된 인증서는 kube-apiserver에서 client 인증서로 인정되어야 한다. CA bundle은 다른 수단으로 배포되지 않는다.
  - Permitted subjects: subject 제한 없음. 그러나 approver, signer는 승인이나 서명을 거부할 수 있다. cluster 관리자 수준의 사용자나 그룹과 같은 특정 subject는 k8s 방식에 따라 다를 수 있지만, 승인 및 서명 전에 추가적인 검토가 필요하다. CertificateSubjectRestriction admission 플러그인은 기본적으로 `system:master`를 제한한다. 그러나 cluster 관리자 subject가 여러개일 수 있다.
  - Permitted x509 extensions - subjectAltName, key usage extension을 준수하고 다른 extension은 버린다.
  - Permitted key usages: ["client auth"]를 포함해야 한다. ["digital signature", "key encipherment", "client auth"] 이외의 key usage는 포함하지 않는다.
  - Expiration/certificate lifetime: 해당 signer에 대한 kube-controller-manager의 구현은 `--cluster-signing-duration` flag 값을 사용하며 `.spec.expirationSeconds`가 지정된 경우에는 두 값 중 최소 값을 사용한다.
  - CA bit allowed/disallowed: 허용되지 않음
- `kubernetes.io/kube-apiserver-client-kubelet`: signs client certificates that will be honored as client certificates by the API server. May be auto-approved by kube-controller-manager. 
- `kubernetes.io/kubelet-serving`: signs serving certificates that are honored as a valid kubelet serving certificate by the API server, but has no other guarantees. Never auto-approved by kube-controller-manager.
- `kubernetes.io/legacy-unknown`: has no guarantees for trust at all. Some third-party distributions of Kubernetes may honor client certificates signed by it. The stable CertificateSigningRequest API (version certificates.k8s.io/v1 and later) does not allow to set the signerName as kubernetes.io/legacy-unknown. Never auto-approved by kube-controller-manager.

kube-controller-manager는 각 내장 signer에 대해 control plane signer를 구현한다. Failures for all of these are only reported in kube-controller-manager logs.

> **Note**:  
> `spec.expirationSeconds` 필드는 k8s v1.22에서 추가됐다. k8s의 이전 버전은 이 필드를 무시한다. v1.22 이전 버전의 kube-apiserver는 이 필드를 무시한다.

Distribution of trust happens out of band for these signers. Any trust outside of those described above are strictly coincidental. For instance, some distributions may honor kubernetes.io/legacy-unknown as client certificates for the kube-apiserver, but this is not a standard. None of these usages are related to ServiceAccount token secrets .data[ca.crt] in any way. That CA bundle is only guaranteed to verify a connection to the API server using the default service (kubernetes.default.svc).

### Custom signers
custom signer를 사용할 수도 있습니다. custom signer는 유사한 접두사 이름을 가져야 하지만 사용자 고유 도메인 이름을 사용해야 한다. 예를 들어, open-fictional.example 도메인을 사용하는 오픈 소스 프로젝트를 대표한다면, `issuer.open-fictional.example/service-mesh`와 같은 signer 이름을 사용할 수 있다.

custom signer는 k8s API를 사용하여 인증서를 발급한다. 관련 내용은 API-based signers를 참고한다.

## Signing
### Control plane signer
k8s control plane의 kube-controller-manager는 내장 signer을 구현한다.

> **Note**:  
> k8s v1.18 이전 버전에서는 승인된 모든 csr에 서명을 수행했다.

> **Note**:  
> `spec.expirationSeconds` 필드는 k8s v1.22에서 추가됐다. k8s의 이전 버전은 이 필드를 무시한다. v1.22 이전 버전의 kube-apiserver는 이 필드를 무시한다.

### API-based signers

## Approval or rejection
signer가 csr에 대해 인증서를 발급하기 전에 일반적으로 승인이 됐는지 먼저 확인한다.

### Control plane automated approval
kube-controller-manager는 `kubernetes.io/kube-apiserver-client-kubelet` signerName을 갖는 csr에 대한 내장 approver를 제공한다. 해당 내장 approver는 csr에 대한 다양한 권한을 no의 credential([TLS bootstrapping](https://kubernetes.io/docs/reference/access-authn-authz/kubelet-tls-bootstrapping/#approval) 참고)에 위임한다. kube-controller-manager는 SubjectAccessReview resource를 사용해 인증서 승인을 위한 인가를 확인한다.

### Approval or rejection using kubectl
k8s 관리자(적절한 권한이 있는)는 kubectl을 사용해 csr를 승인, 거절할 수 있다.

CSR 승인은 다음과 같다.
``` sh
kubectl certificate approve <certificate-signing-request-name>
```

CSR 거절은 다음과 같다.
``` sh
kubectl certificate deny <certificate-signing-request-name>
```

### Approval or rejection using the Kubernetes API

## Cluster trust bundles

## How to issue a certificate for a user
