이 페이지는 admission controller에 대한 개요를 제공한다.

admission controller는 resource가 저장되기 전, 요청이 authenticated, authorized된 후 kube-apiserver로 들어오는 요청을 가로채는 코드 조각이다.

k8s의 여러 중요 기능들은 해당 기능을 올바르게 지원하기 위해 admission controller가 활성화되어야 한다. 결과적으로, 올바른 admission controller 세트로 적절하게 구성되지 않은 kube-apiserver는 기대하는 모든 기능을 지원하지 못하는 불완전한 서버다.

## What are they?
admission controller는 k8s kube-apiserver 내의 코드로, resource를 수정하려는 요청에 포함된 데이터를 확인한다.

admission controller는 object를 생성(create), 삭제(delete) 또는 수정(modify)하는 요청에 적용된다. 또한 admission controller는 kube-apiserver proxy를 통해 po에 연결하는 요청과 같은 사용자 정의 동사(custom verbs)도 차단할 수 있다. admission controller는 object를 읽는(get, watch 또는 list) 요청은 차단하지 않는다(차단할 수 없다). 왜냐하면 읽기 작업은 admission control 계층을 거치지 않기 때문이다.

admission control 메커니즘은 validating(검증) 또는 mutating(변경) 또는 둘 다일 수 있다. mutating controller는 수정되는 resource의 데이터를 변경할 수 있지만, validating controller는 그럴 수 없다.

k8s 1.33 버전의 admission controller 전체 목록은 아래에서 살펴볼 것이다. 이러한 controller 목록은 모두 kube-apiserver 바이너리에 컴파일되어 포함되고 오직 cluster 관리자만이 설정할 수 있다.

### Admission control extension points
전체 목록에는 세 가지 특별한 admission controllers가 있다: MutatingAdmissionWebhook, ValidatingAdmissionWebhook, ValidatingAdmissionPolicy. 앞의 두 webhook admission controller는 각각 설정된 mutating, validating admission control webhook을 실행한다. ValidatingAdmissionPolicy는 외부 HTTP 호출에 의존하지 않고 API 내에 선언적 검증 코드를 포함할 수 있는 방법을 제공한다.

이 세 가지 admission controller를 사용하여 admission 시점에 cluster 동작을 사용자가 커스텀할 수 있다.

### Admission control phases
admission control 프로세스는 두 phase로 진행된다.
1. 첫 번째 phase에서는 mutating admission controller가 실행된다.
2. 두 번째 phase에서는 validating admission controller가 실행된다.

일부 admission controller는 두 가지 역할을 모두 수행한다는 점에 유의해야 한다. 어느 단계에서든 admission controller 중 하나라도 요청을 reject하면 전체 요청은 즉시 reject되고 최종 사용자에게 오류가 반환된다.

마지막으로, 해당 object를 변경하는 것 뿐만 아니라 admission controller는 때때로 관련된 resource를 변경하는 side effect를 가질 수 있다. 예를 들어 quota 사용량이 대표적인 예시다.
1. 사용자가 cpu 1, 메모리 1GiB를 사용하는 po를 생성 요청을 수행한다.
2. quota를 담당하는 admission controller에서 quota 사용량을 확인하고 충분한 경우 현재 사용량을 늘린다(side effect).
3. 이후 admission controller에서 예를 들어, 보안 규정에 어긋나 reject를 수행한다.

결과적으로 quota 사용량은 증가했지만 po는 생성되지 않은 상태가 되며 복구(reclamation) 또는 조정(reconciliation) 프로세스가 필요하다. 왜냐하면 특정 admission controller는 해당 요청이 다른 모든 admission controller를 통과할 것이라고 확신할 수 없기 때문이다.

아래는 호출 순서 다이어그램이다.

![](https://kubernetes.io/docs/reference/access-authn-authz/admission-control-phases.svg)

## Why do I need them?
k8s의 여러 중요 기능들은 해당 기능을 올바르게 지원하기 위해 admission controller가 활성화되어야 한다. 결과적으로, 올바른 admission controller 세트로 적절하게 구성되지 않은 kube-apiserver는 불완전한 서버이며 기대하는 모든 기능을 지원하지 않을 수 있다.

## How do I turn on an admission controller?
kube-apiserver의 `--enable-admission-plugins` flag에 쉼표로 구분된 목록을 명시해 admission controller plugin를 활성화할 수 있다. 예를 들어, 다음 명령줄은 NamespaceLifecycle, LimitRanger admission control plugin를 활성화한다.
``` sh
kube-apiserver --enable-admission-plugins=NamespaceLifecycle,LimitRanger ...
```

> **Note:**  
> k8s cluster가 배포된 방식과 kube-apiserver가 시작되는 방식에 따라 설정을 다른 방식으로 적용해야 할 수 있다. 예를 들어, kube-apiserver가 systemd 서비스로 배포된 경우 systemd 유닛 파일을 수정해야 할 수 있으며, k8s가 자체 호스팅 방식(self-hosted way)으로 배포된 경우 kube-apiserver의 매니페스트 파일을 수정해야 할 수도 있다.

## How do I turn off an admission controller?
kube-apiserver의 `--disable-admission-plugins` flag에 쉼표로 구분된 목록을 명시해 admission control plugin을 비활성화할 수 있다. 이는 `--enable-admission-plugins`보다 우선 순위가 높다.
``` sh
kube-apiserver --disable-admission-plugins=PodNodeSelector,AlwaysDeny ...
```

## Which plugins are enabled by default?
어떤 admission plugin이 활성화되어 있는지 확인하기 위해 아래 명령어를 실행한다.
``` sh
kube-apiserver -h | grep enable-admission-plugins
```

k8s 1.33 버전 기준 아래는 기본 결과다.
``` sh
CertificateApproval, CertificateSigning, CertificateSubjectRestriction, DefaultIngressClass, DefaultStorageClass, DefaultTolerationSeconds, LimitRanger, MutatingAdmissionWebhook, NamespaceLifecycle, PersistentVolumeClaimResize, PodSecurity, Priority, ResourceQuota, RuntimeClass, ServiceAccount, StorageObjectInUseProtection, TaintNodesByCondition, ValidatingAdmissionPolicy, ValidatingAdmissionWebhook
```

## What does each admission controller do?

### MutatingAdmissionWebhook
Type: Mutating

이 admission controller는 요청과 매칭되는 mutating webhook을 모두 호출한다. 매칭되는 webhook은 순차적으로 호출되며, 각 webhook은 필요한 경우 object를 수정할 수도 있다.

이 admission controller는 mutating phase에서만 실행된다.

호출된 webhook이 side effect을 가지는 경우(예를 들어, quota 감소), 반드시 reconciliation 시스템을 갖춰야 한다. 왜냐하면 해당 webhook 실행이 성공하더라도 이후 webhook, validating admission controller가 해당 요청을 reject할 수도 있기 때문이다.

MutatingAdmissionWebhook을 비활성화하는 경우, `--runtime-config` flag를 사용해 `admissionregistration.k8s.io/v1` 그룹/버전의 MutatingWebhookConfiguration API도 비활성화해야 한다. 기본적으로는 모두 활성화되어 있다.

#### Use caution when authoring and installing mutating webhooks
- 사용자가 생성하려고 시도한 object가 실제로 반환된 object5와 다를 때 혼란을 겪을 수 있다.
- 내장된 control loop는 생성하려고 시도한 object를 다시 읽었을 때 해당 object가 변경된 경우 중단될 수 있다.
    - 원래 요청에서 설정된 필드를 덮어쓰는 것보다 초기에 설정되지 않은 필드를 설정하는 것이 문제를 일으킬 가능성이 적다. 후자의 작업을 피하는 것을 권장한다.
- 내장 resource나 third-party resource의 control loop에 대한 향후 변경 사항으로 인해 현재 잘 작동하는 webhook이 중단될 수 있다. webhook instsallation API가 성공적이더라도 모든 가능한 webhook 동작이 무기한 정상 동작할 것이라고 보장되지는 않는다.

### ValidatingAdmissionPolicy
Type: Validating

이 admission controller는 매칭되는 요청에 대해 `CEL(Common Expression Language)` 검증을 구현한다. validatingadmissionpolicy feature gate와 `admissionregistration.k8s.io/v1alpha1` 그룹/버전 API가 모두 활성화될 때 사용 가능하다. 만약 ValidatingAdmissionPolicy 중 하나라도 실패하면 요청은 실패한다.

### ValidatingAdmissionWebhook
Type: Validating

이 admission controller는 요청과 매칭되는 모든 validating webhook을 호출한다. 매칭하는 webhook는 병렬로 호출되며 하나라도 요청을 거부하면 해당 요청은 실패한다. 이 admission controller는 validation 단계에서만 실행된다. MutatingAdmissionWebhook admission controller에 의해 호출되는 webhook와는 달리 이 controller가 호출하는 webhook는 object를 mutating할 수 없다.

호출된 webhook이 side effect을 가지는 경우(예를 들어, quota 감소), 반드시 reconciliation 시스템을 갖춰야 한다. 왜냐하면 해당 webhook 실행이 성공하더라도 이후 webhook, validating admission controller가 해당 요청을 reject할 수도 있기 때문이다.

ValidatingAdmissionWebhook을 비활성화하는 경우, `--runtime-config` flag를 통해 `admissionregistration.k8s.io/v1` 그룹/버전의 ValidatingWebhookConfiguration API도 비활성화해야 한다.

## Is there a recommended set of admission controllers to use?
권장되는 admission controller는 기본적으로 활성화되어 있으므로([목록](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/#options)), 명시적으로 지정할 필요가 없다. `--enable-admission-plugins` flag를 사용해 기본 세트 외에 추가적인 admission controller를 활성화할 수 있다(순서는 중요하지 않음).