pod security standrad는 보안 스펙트럼을 광범위하게 다루기 위해 3가지 다른 정책을 정의한다. 이러한 정책은 누적되며 매우 허용적인 것부터 매우 제한적인 것까지 다양하다. 여기서는 각 정책의 필수 조건을 간략하게 설명한다.

|Profile|Description|
|-------|-----------|
|Privileged|가능한 한 가장 광범위한 권한 레벨을 제공하는 제한이 없는 정책. 이 정책은 알려진한 권한 상승을 허용한다.|
|Baseline|알려진 권한 상승을 제한하는 최소한의 정책. Allows the default (minimally specified) Pod configuration.|
|Restricted|현재 po의 보안의 모든 사항을 엄격하게 따르는 제한된 정책|

## Profile Details
### Privileged
privileged 정책은 보안에 대한 제한 사항이 없다. 이러한 유형의 정책은 일반적으로 신뢰할 수 있는 사용자가 관리하는 system-, infrastructure- 레벨 workload를 대상으로 한다.

privileged 정책은 제한이 없는 것으로 정의된다. 기본적으로 allow-by-default 메커니즘(예를 들어 gatekeeper)는 기본적으로 privilged다. 반대로 deny-by-default 메커니즘(예를 들어 psp)의 경우 기본적으로 모든 제한을 비활성화해야 한다.

### Baseline
baseline 정책은 알려진 권한 상승을 방지하면서 일반적인 container화 된 workload에 대한 채택을 용이하게 하는 것을 목표로한다. 이 정책은 중요하지 않은 애플리케이션 운영자 및 개발자를 대상으로 한다. 아래 목록에 있는 제어는 금지되어야 한다:

**Note**: 아래 표에서 와일드카드(*)는 목록의 모든 요소를 나타낸다. 예를 들어 spec.containers[\*].securityContext은 정의된 모든 container에 대한 security context 객체를 참조한다. 나열된 container 중 하나라도 요구 사항을 충족하지 못하면 전체 po에 대한 유효성 검사에 실패한다.

### Restricted
restricted 정책은 일부 호환성을 희생하면서 현재 po 보안 강화 best practice를 실천하는 것을 목표로 한다. 보안이 중요한 애플리케이션의 운영자 및 개발자는 물론 신뢰도가 낮은 사용자를 대상으로 한다. 아래 목록에 있는 제어는 금지되어야 한다:

**Note**: 아래 표에서 와일드카드(*)는 목록의 모든 요소를 나타낸다. 예를 들어 spec.containers[\*].securityContext은 정의된 모든 container에 대한 security context 객체를 참조한다. 나열된 container 중 하나라도 요구 사항을 충족하지 못하면 전체 po에 대한 유효성 검사에 실패한다.

## Policy Instantiation

### Alternatives

## Pod OS field

### Restricted Pod Security Standard changes

## FAQ
### Why isn't there a profile between privileged and baseline? 

### What's the difference between a security profile and a security context?
security context는 실행 시 po, container를 설정한다. 이는 po manifest에 po 또는 container에 대해 정의되며 container runtime에 대한 매개변수를 나타낸다.

security profile은 security context에 대한 설정, security context 외 관련 파라미터에 대해 적용되는 control plane 메커니즘이다. 2021년 7월부터 내장 pod security admission controller로 인해 psp는 더 이상 사용되지 않는다.
### What about sandboxed Pods?