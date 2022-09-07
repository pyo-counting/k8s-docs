k8s pod security standard는 po에 대해 다양한 격리 수준을 정의한다. 이러한 표준을 사용하면 명확하고 일관된 방식으로 po의 동작을 제한하는 방법을 정의할 수 있다.

k8s는 pod security standard를 위해 pod security admission controller를 제공한다. po 보안 제한은 po가 생성될 때 ns 수준에서 동작한다.

### Built-in Pod Security admission enforcement
이 기능은 k8s v1.25부터 stable하다.

## Pod Security levels
pod security admission은 po의 security context와 pod security standrad에 의해 정의된 3가지 레벨(`privileged`, `baseline`, `restricted`)에 따른 관련 필드에 대한 요구 사항을 지정한다.

## Pod Security Admission labels for namespaces
기능이 활성화되거나 웹훅이 설치되면 각 ns마다 po 보안에 사용할 admission control mode를 정의하도록 ns를 설정할 수 있다. k8s는 사전 정의된 pod security standrad 중 ns에 사용할 레벨을 정의하기 위해 설정할 수 있는 레이블 집합을 정의한다. 사용한 label은 잠재적 위반이 감지되는 경우 control plane이 취할 조치를 정의한다:

|Mode|Description|
|----|-----------|
|enforce|장책 위반은 po가 거절되도록 한다.|
|audit|정책 위반은 audit log에 기록된 이벤트에 audit annotation 추가를 트리거하지만 허용은한다.|
|warn|정책 위반은 사용자에게 경고를 표시하지만 허용은한다.|

ns는 일부 또는 모드를 설정하거나 다른 모드에 대해 다른 수준을 설정할 수 있다.

각 모드에 사용되는 정책을 결정하는 두 label이 있다.

``` yaml
# The per-mode level label indicates which policy level to apply for the mode.
#
# MODE must be one of `enforce`, `audit`, or `warn`.
# LEVEL must be one of `privileged`, `baseline`, or `restricted`.
pod-security.kubernetes.io/<MODE>: <LEVEL>

# Optional: per-mode version label that can be used to pin the policy to the
# version that shipped with a given Kubernetes minor version (for example v1.25).
#
# MODE must be one of `enforce`, `audit`, or `warn`.
# VERSION must be a valid Kubernetes minor version, or `latest`.
pod-security.kubernetes.io/<MODE>-version: <VERSION>
```

## Workload resources and Pod templates
po는 deploy, job과 같은 workload 객체를 통해 간접적으로 생성되는 경우가 많다. workload 객체는 po template을 정의하고 해당 workload resource와 관련있는 controller는 해당 template을 기반으로 po를 생성한다. 위반을 조기에 발견할 수 있도록 audit, warning 모드가 workload resource에 적용된다. 하지만 enforce 모드는 workload resource에 적용되지 않고 po에 적용된다.

## Exemptions
ns와 관련있는 정책으로 인해 제한된 po 생성을 허용하기 위해 po 보안 시행에 대한 예외를 정의할 수 있다. 예외는 admission controller 설정에서 정적으로 구성할 수 있다.

예외는 명시적으로 나열되어야 한다. 예외 기준을 만족하는 요청은 admission controller에서 무시(모든 enforce, audit, warn에 대한 동작은 생략됨)된다. 예외에 대한 기준은 아래가 있다:

- Usernames: 예외처리된 인증(authenticated) 사용자 이름을 가진 사용자의 요청은 무시된다.
- RuntimeClassNames: 예외 runtime class를 지정하는 po 및 workload resource는 무시된다.
- Namespaces: 예외 ns의 po, workload resource는 무시된다.

**Caution**: 대부분의 po는 workload resource에 대한 응답으로 controller에서 생성된다. 즉 Usernames를 사용해 사용ㅈ아를 제외하면 po를 직접 생성할 때만 적용되어 workload resource를 생성할 때는 이 제외가 적용되지 않는다. controller sa(예를 들어 system:serviceaccount:kube-system:replicaset-controller)는 일반적으로 예외 대상이 되면 안된다. 왜냐하면 해당 worklaod resource를 생성할 수 있는 모든 사용자에 대해 암묵적으로 예외 처리가 되기 때문이다.

다음 po 필드에 대한 업데이트는 정책 검사에서 제외된다. 즉 po 업데이트 요청이 아래 필드만 변경하는 경우 po가 현재 정책 수준을 위반하더라도 거절되지 않는다.

- seccomp, AppArmor annotation에 대한 변경 사항을 제외한 모든 .metadata 업데이트:
    - seccomp.security.alpha.kubernetes.io/pod (deprecated)
    - container.seccomp.security.alpha.kubernetes.io/* (deprecated)
    - container.apparmor.security.beta.kubernetes.io/*
- .spec.activeDeadlineSeconds
- .spec.tolerations
