[compiled-in admission plugins](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/) 외에도, admission plugin은 확장 기능으로 개발되어 런타임에 구성되는 webhook으로 실행될 수 있다. 이 페이지에서는 admission webhook을 빌드, 구성, 사용 및 모니터링하는 방법을 설명한다.

## What are admission webhooks?
admission webhook는 admission 요청을 수신 및 처리하는 HTTP callback이다. 두 가지 유형의 admission webhook를 정의할 수 있다: validating admission webhook, mutating admission webhook. mutating admission webhook이 먼저 호출되며, kube-apiserver로 전송된 객체를 수정해 사용자 정의 기본값을 적용할 수 있다. 모든 객체 수정이 완료되면 kube-apiserver에 의해 검증된 후, validating admission webhook이 호출되어 사용자 정의 정책을 기반으로 요청을 거부할 수 있다.

> **Note**:  
> 정책을 적용하기 위해 객체의 최종 상태를 확인해야 하는 admission webhooks는 검증(validating) admission webhook을 사용해야 합니다. 왜냐하면 객체는 변경(mutating) webhooks에 의해 처리된 후에도 수정될 수 있기 때문입니다.