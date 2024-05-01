## What are service accounts?
sa는 k8s에서 고유한 identity를 제공하는 non-human identity의 한 종류다. po, 시스템 구성 요소 등은 특정 sa의 credential을 사용해 sa로 인증 받을 수 있다. 이 identity는 kube-apiserver에 대한 authentication, identity-based security policy를 포함한 다양한 상황에서 유용하다.

sa는 kube-apiserver에 object로 존재하며 아래 특징을 갖는다.
- Namespaced: 각 sa는 k8s ns에 속한다. 각 ns은 생성 시 default sa를 갖는다.
- Lightweight: sa는 k8s API를 통해 정의할 수 있으며 필요에 따라 언제든지 생성할 수 있다.
- Portable: A configuration bundle for a complex containerized workload might include service account definitions for the system's components. The lightweight nature of service accounts and the namespaced identities make the configurations portable.

sa는 cluster 내에서 사용자 계정(authenticated human user)와는 다르다. 기본적으로 사용자 계정은 kube-apiserver에 존재하지 않으며, 대신 사용자 identity를 opaque 데이터로 취급한다. 여러 방법을 사용해 사용자 계정을 인증할 수 있으며 일부 k8s distribution은 kube-apiserver에 사용자를 나타내기 위해 custom extension API를 추가한다.
| Description    | ServiceAccount                                    | User or group                                                      |
|----------------|---------------------------------------------------|--------------------------------------------------------------------|
| Location       | Kubernetes API (ServiceAccount object)            | External                                                           |
| Access control | Kubernetes RBAC or other authorization mechanisms | Kubernetes RBAC or other identity and access management mechanisms |
| Intended use   | Workloads, automation                             | People                                                             |

### Default service accounts
cluster 생성 시 k8s는 cluster의 모든 ns에 default 라는 이름의 sa object를 자동으로 생성한다. 각 ns의 default sa는 rbac가 사용되도록 설정된 경우 k8s가 authenticated principal에게 부여하는 [default API discovery permissions](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#default-roles-and-role-bindings) 외에는 기본적으로 권한을 얻지 못한다. default sa를 삭제하면 control plane은 다시 생성한다.

ns에 po를 배포하고 po에 sa를 명시적으로 할당하지 않으면([manually assign ServiceAccount to the Pod](https://kubernetes.io/docs/concepts/security/service-accounts/#assign-to-pod)) k8s는 해당 ns의 default sa를 po에 할당한다.

## Use cases for Kubernetes service accounts
아래와 같은 시나리오에서 sa를 사용할 수 있다.
- 아래와 같이 po가 kube-apiserver와 통신이 필요할 경우
    - secret에 대한 read-only 접근 제공
    - po가 `kube-node-lease` ns의 lease object를 read, list, watch할 수 있도록 cross-namespace access 제공
- po가 외부 서비스와 통신이 필요한 경우. For example, a workload Pod requires an identity for a commercially available cloud API, and the commercial provider allows configuring a suitable trust relationship.
- [Authenticating to a private image registry using an imagePullSecret](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#add-imagepullsecrets-to-a-service-account)
- 외부 서비스가 kube-apiserver와 통신이 필요할 때. 예를 들어 CI/CD pipeline의 일부로 cluster에 authentication이 필요
- You use third-party security software in your cluster that relies on the ServiceAccount identity of different Pods to group those Pods into different contexts.

## How to use service accounts
k8s sa를 사용하기 위해 아래 작업을 수행해야 한다.
1. sa object를 생성한다.
2. [rbac](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)와 같은 authorization을 사용해 sa에 권한을 부여한다.
3. po에 sa를 할당한다.

### Grant permissions to a ServiceAccount
각 sa가 필요로하는 최소 권한을 부여하기 위해 내장 k8s [role-based access control(RBAC)](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)를 사용할 수 있다. role을 생성해 권한을 부여하고 role과 sa를 bind할 수 있다. rbac을 사용해 sa 권한이 최소 권한 원칙(principle of least privileage)을 따르도록 최소 권한 집합을 정의할 수 있다. 해당 sa를 사용하는 po는 더 많은 권한을 얻지 못한다. 

#### Cross-namespace access using a ServiceAccount
rbac를 사용해 한 ns의 sa가 cluster의 다른 ns에 있는 resource에 대한 작업을 수행할 수 있다. 예를 들어, dev ns에 sa와 po가 있고 po가 maintenance ns에 실행 중인 job을 볼 수 있도록 할 수 있다. job object를 나열(list)할 수 있는 권한을 부여하는 role object를 만들 수 있다. 그런 다음 maintenance ns에서 rolebinding object를 생성해 해당 role을 sa object에 binding할 수 있다. 이제 dev ns의 po가 해당 sa를 사용해 maintenance ns의 job object을 나열할 수 있다.

### Assign a ServiceAccount to a Pod
po에 sa를 할당하려면 po의 `.spec.serviceAccountName` 필드를 사용한다. k8s는 해당 sa에 대한 credential을 자동으로 po에 제공한다. v1.22부터 k8s는 `TokenRequest` API를 사용해 짧은 수명의 **automatically rotating token**을 얻고 token을 [projected volume](https://kubernetes.io/docs/concepts/storage/projected-volumes/#serviceaccounttoken)으로 mount한다.
``` yaml
spec:
  volumes:
    - name: kube-api-access-jjwsm
      projected:
        sources:
          - serviceAccountToken:
              expirationSeconds: 3607
              path: token
          - configMap:
              name: kube-root-ca.crt
              items:
                - key: ca.crt
                  path: ca.crt
          - downwardAPI:
              items:
                - path: namespace
                  fieldRef:
                    apiVersion: v1
                    fieldPath: metadata.namespace
        defaultMode: 420
  containers:
    (...생략...)
      volumeMounts:
        - name: kube-api-access-jjwsm
          readOnly: true
          mountPath: /var/run/secrets/kubernetes.io/serviceaccount
```

기본적으로 k8s는 할당된 sa(사용자가 지정한 sa 또는 default sa)에 대한 credential을 po에 제공한다.

k8s가 sa에 대한 credential을 자동으로 주입(inject)하지 못하도록 하기 위해 po에 `.spec.automountServiceAccountToken`을 false로 설정한다.

1.22 이전 버전에서는 긴 수명의 static token을 secret으로 제공한다.

#### Manually retrieve ServiceAccount credentials
sa에 대한 credential을 기본 mount 위치가 아니거나, kube-apiserver가 아닌 audience에 필요한 경우 아래 방법을 사용한다.
- [TokenRequest API](https://kubernetes.io/docs/reference/kubernetes-api/authentication-resources/token-request-v1/): (권장) 애플리케이션 코드에서 짧은 수명의 sa token을 요청한다. token은 자동으로 만료되며 만료시 rotate할 수 있다. k8s를 모르는 legacy 애플리케이션의 경우 po 내 sidecar container를 이용해 token을 대신 발급 받을 수 있다.
- [Token Volume Projection](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#serviceaccount-token-volume-projection): (권장) k8s v1.20부터 sa token을 po에 projected volume으로 추가한다. projected token은 자동으로 만료되고 kubelet은 만료되기 전에 token을 rotate한다.
- [Service Account Token Secets](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#manually-create-an-api-token-for-a-serviceaccount): (권장하지 않음)

> **Note**:  
> k8s cluster 외부 애플리케이션을 위해 긴 수명의 sa token을 고려할 수도 있다. This allows authentication, but the Kubernetes project recommends you avoid this approach. Long-lived bearer tokens represent a security risk as, once disclosed, the token can be misused. Instead, consider using an alternative. For example, your external application can authenticate using a well-protected private key and a certificate, or using a custom mechanism such as an authentication webhook that you implement yourself.
>
> You can also use TokenRequest to obtain short-lived tokens for your external application.

### Restricting access to Secrets
sa에 `kubernetes.io/enforce-mountable-secrets` annotation을 추가할 수 있다. 이 annotation이 적용되면 해당 sa을 사용할 때 특정 secret만 사용할 수 있다.

아래는 예시다.
``` yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    kubernetes.io/enforce-mountable-secrets: "true"
  name: my-serviceaccount
  namespace: my-namespace
imagePullSecrets:
(...생략...)
secrets:
(...생략...)
```

해당 annotation이 true로 설정되면 k8s control plane은 sa의 secret에 대한 mount를 제한한다.
1. po의 sa `.secrets` 필드에 나열된 secret만 po에서 mount할 수 있다.
2. po의 sa `.secrets` 필드에 나열된 secret만 po의 `envFrom`에서 참조할 수 있다.
3. po의 sa `.secrets` 필드에 나열된 secret만 po의 `imagePullSecrets`에서 참조할 수 있다.

cluster 관리자는 이러한 restrinction을 이해하고 적용함으로써 보다 엄격한 보안을 유지할 수 있다.

## Authenticating service account credentials

### Authenticating service account credentials in your own code
### Alternatives