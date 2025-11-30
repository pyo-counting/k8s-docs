pod security standrad는 보안 스펙트럼을 광범위하게 다루기 위해 3가지 다른 정책을 정의한다. 이러한 정책은 누적되며 매우 허용적인 것부터 매우 제한적인 것까지 다양하다. 여기서는 각 정책의 필수 조건을 간략하게 설명한다.

|Profile|Description|
|-------|-----------|
|Privileged|가능한 한 가장 광범위한 권한 레벨을 제공하는 제한이 없는 정책. 이 정책은 known privilege escalation을 허용한다.|
|Baseline|privilege escalations을 제한하는 최소한의 정책. Allows the default (minimally specified) Pod configuration.|
|Restricted|po의 hardening best practices을 엄격하게 따르는 제한된 정책|

## Profile Details
### Privileged
privileged 정책은 보안에 대한 제한 사항이 없다. 이러한 유형의 정책은 일반적으로 신뢰할 수 있는 사용자가 관리하는 system-, infrastructure- 레벨 workload를 대상으로 한다.

privileged 정책은 제한이 없다.

### Baseline
baseline 정책은 known privilege escalation을 방지하면서 일반적인 container화 된 workload에 적용을 목표로한다. 이 정책은 일반적인 애플리케이션을 위한 것이다. 아래는 정책 목록이다.

> **Note**:  
> 아래 표에서 와일드카드(*)는 목록의 모든 요소를 나타낸다. 예를 들어 `spec.containers[*].securityContext`은 정의된 모든 container에 대한 security context 객체를 참조한다. 나열된 container 중 하나라도 요구 사항을 충족하지 못하면 전체 po에 대한 유효성 검사에 실패한다.

### Restricted
restricted 정책은 일부 호환성을 희생하면서 현재 po 보안 강화 best practice를 실천하는 것을 목표로 한다. 보안이 중요한 애플리케이션의 운영자 및 개발자는 물론 신뢰도가 낮은 사용자를 대상으로 한다. 아래는 정책 목록이다.

> **Note**:  
> 아래 표에서 와일드카드(*)는 목록의 모든 요소를 나타낸다. 예를 들어 `spec.containers[*].securityContext`은 정의된 모든 container에 대한 security context 객체를 참조한다. 나열된 container 중 하나라도 요구 사항을 충족하지 못하면 전체 po에 대한 유효성 검사에 실패한다.

## Policy Instantiation
정책 정의를 정책 인스턴스화와 분리하면, 기본 실행 메커니즘과 독립적으로 cluster 전체에서 정책에 대한 공통적인 이해와 일관된 언어를 사용할 수 있다.

메커니즘이 성숙해짐에 따라, 이들은 아래에서 정책별로 정의될 것이다. 개별 정책의 실행 방법(시행 방식)은 여기에서 정의되지 않으며 아래 링크를 참고한다.

### Alternatives
k8s 생태계에서 개발된 정책 적용 프로젝트는 다음과 같다.
- Kubewarden
- Kyverno
- OPA Gatekeeper

## Pod OS field
k8s는 linux, window OS 환경에서 no 실행을 지원한다. 두 OS 환경의 no를 동일 cluster에서 구성할 수 있다. 하지만 window 환경에서의 linux 환경과 비교해 차이점 및 제한 사항이 있다. 대부분의 경우 po의 securityContext가 동작하지 않는다.

### Restricted Pod Security Standard changes

#### OS-specific policy controls
아래 제어 항목은 `.spec.os.name` 필드가 windows가 아닐 경우에만 필수다.
-Privilege Escalation
- Seccomp
- Linux Capabilities

## User namespaces
user namespace는 격리를 위해 linux에서만 제공하는 기능이다. pss와 같이 어떻게 동작하는지는 [documentation](https://kubernetes.io/docs/concepts/workloads/pods/user-namespaces/#integration-with-pod-security-admission-checks)을 참고한다.

## FAQ
### Why isn't there a profile between privileged and baseline?

### What's the difference between a security profile and a security context?
security context는 런타임 시점에 po, container를 설정한다. 이는 po manifest에 po 또는 container에 대해 정의되며 container runtime에 대한 매개변수를 나타낸다.

security profile은 security context에 대한 설정 뿐만아니라 security context 외 관련 파라미터에 대해 적용되는 control plane 관점에서의 메커니즘이다. 2021년 7월부터 내장 pod security admission controller로 인해 psp는 더 이상 사용되지 않는다.

### What about sandboxed Pods?