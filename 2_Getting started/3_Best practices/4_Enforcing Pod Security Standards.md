## Using the built-in Pod Security Admission Controller
psa controller는 psp를 대체한다.

### Configure all cluster namespaces
아무런 설정도 없는 ns는 클러스터 보안 모델의 중대한 허점으로 간주되어야 한다. 각 ns에서 실행되는 workload의 유형을 분석하는 시간을 갖고, pss를 참조해 각 ns에 적절한 보안 수준을 결정할 것을 권장한다. label이 지정되지 않은 ns는 단순히 아직 평가되지 않았음을 의미할 뿐이어야 한다.

만약 모든 ns의 모든 workload가 동일한 보안 요구 사항을 가진 시나리오일 때 psa label을 일괄적으로 적용하는 방법은 [exmaple](https://kubernetes.io/docs/tasks/configure-pod-container/enforce-standards-namespace-labels/#applying-to-all-namespaces)을 참고한다.

### Embrace the principle of least privilege
이상적인 환경에서는 모든 ns의 모든 po가 Restricted 정책의 요구 사항을 충족하는 것이다. 하지만 일부 workload는 높은 권한을 필요로 하기 때문에  가능하지도, 실용적이지도 않다. Privileged workload를 허용하는 ns는 적절한 접근 제어를 수립해야 한다. 그리고 해당 ns에서 실행되는 workload들에 대해서는 보안 요구 사항에 대한 문서를 유지 관리해야 한다. 가능하다면 요구 사항들을 어떻게 더 제한할 수 있을지(제약할 수 있을지) 계속 고민해야 한다.

### Adopt a multi-mode strategy
psa controller의 audit, warn mode는 기존 workload를 중단시키지 않으면서 po에 대한 중요한 보안 통찰력을 쉽게 수집할 수 있도록 도와준다.

모든 ns에 대해 궁극적으로 enforce mode를 사용하고자 할 경우 적용할 정책 level과 k8s 버전에 대해 두 mode를 적용하면 경고 및 audit annotation을 통해 위반 사항을 모니터링할 수 있다. workload 작성자(개발자)가 원하는 level에 도달하기 위해 변경 사항을 적용할 것으로 예상된다면, warn mode를 활성화한다. audit log를 사용해 원하는 level에 맞추기 위한 변경 사항을 모니터링/추진할 것으로 예상된다면 audit mode를 활성화하한다.

enforce mode가 원하는 값으로 설정된 후에도, 이 mode들은 몇 가지 다른 방식으로 여전히 유용할 수 있다.
- warn을 enforce와 동일한 정책 level에 설정하면 사용자는 유효성 검사를 통과하지 못하는 po(또는 po 템플릿을 가진 리소스)를 생성하려고 시도할 때 경고를 받을 수 있다. 이는 그들이 해당 리소스를 업데이트하여 규정을 준수하게 하는 데 도움을 준다.
- enforce를 특정 최신 버전이 아닌 버전에 고정(pin)하는 ns에서, audit, warn mode를 enforce와 동일한 정책 level로 설정하되, 최신 버전으로 설정하면 이전 버전에서는 허용되었지만 현재의 모범 사례에 따라서는 허용되지 않는 설정들을 파악할 수 있는 가시성을 제공한다.

## Third-party alternatives
k8s 생태계에서 개발된 정책 적용 프로젝트는 다음과 같다.
- Kubewarden
- Kyverno
- OPA Gatekeeper