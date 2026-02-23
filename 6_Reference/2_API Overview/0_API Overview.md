REST API는 k8s의 기본적인 뼈대(fabric)다. 구성 요소 간의 모든 작업 및 통신, 그리고 외부 사용자 명령은 kube-apiserver가 처리하는 REST API 호출이다. 결과적으로 k8s의 모든 것은 API object로 취급된다.

k8s v1.35 버전에 대한 전체 API는 [Kubernetes API reference](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.35/)을 참고한다.

## API versioning
JSON, Protobuf serialization schema는 schema 변경에 대해 동일한 가이드라인을 따른다. 다음 설명은 두 형식 모두에 적용된다.
- API version 관리와 소프트웨어 버전 관리는 간접적으로 연관되어 있다. API 버전 관리와 소프트웨어 버전 관리 간의 관계에 대한 설명은 [API and release versioning proposal](https://github.com/kubernetes/sig-release/blob/master/release-engineering/versioning.md)을 참고한다.

서로 다른 API version은 서로 다른 level의 안정성 및 지원을 나타낸다. 각 level에 대한 기준의 자세한 정보는 [API Changes documentation](https://git.k8s.io/community/contributors/devel/sig-architecture/api_changes.md#alpha-beta-and-stable-versions)을 참고한다.

각 level에 대한 요약은 다음과 같다.
- alpha
    - version 이름에 alpha가 포함된다(예: `v1alpha1`).
    - 내장된(Built-in) alpha API version은 기본적으로 비활성화되어 있으며 사용하려면 kube-apiserver 설정에서 명시적으로 활성화해야 한다.
    - 소프트웨어에 버그가 있을 수 있다. 기능을 활성화하면 버그가 노출될 수 있다.
    - alpha API에 대한 지원은 예고 없이 언제든지 중단될 수 있다.
    - 이후 소프트웨어 release에서 예고 없이 호환되지 않는 방식으로 API가 변경될 수 있다.
    - 버그 발생 위험이 높고 장기적인 지원이 부족하므로 수명이 짧은 테스트 cluster에서만 사용하는 것이 좋다.
- beta
    - version 이름에 beta가 포함된다 (예: `v2beta3`).
    - 내장된 beta API version은 기본적으로 비활성화되어 있으며, 사용하려면 kube-apiserver 설정에서 명시적으로 활성화해야 한다 (단, k8s 1.22 이전에 도입된 beta version API는 예외로 기본적으로 활성화되어 있다).
    - 내장된 beta API version의 최대 수명은 도입부터 사용 중단(deprecation)까지 9개월 또는 3개의 마이너 release(둘 중 더 긴 쪽)이며, 사용 중단부터 제거(removal)까지 9개월 또는 3개의 마이너 release(둘 중 더 긴 쪽)다.
    - 소프트웨어는 충분히 테스트됐다. 기능을 활성화하는 것은 안전하다고 간주된다.
    - 세부 사항은 변경될 수 있지만, 기능에 대한 지원은 중단되지 않는다.
    - object의 schema, semantics이 후속 beta 또는 stable API version에서 호환되지 않는 방식으로 변경될 수 있다. 이런 경우 마이그레이션 가이드가 제공된다. 후속 beta 또는 stable API version에 맞게 조정하려면 API object를 편집하거나 다시 만들어야 할 수 있으며, 이 과정이 간단하지 않을 수 있다. 마이그레이션으로 인해 해당 기능에 의존하는 애플리케이션에 다운타임이 발생할 수 있다.
    - 프로덕션 용도로 사용하는 것은 권장하지 않는다. 후속 release에서 호환되지 않는 변경 사항이 도입될 수 있다. beta API version이 사용 중단되고 더 이상 제공되지 않으면 후속 beta 또는 stable API version으로 전환해야 한다.
- stable
    - 버전 이름은 vX이며 여기서 X는 정수다.
    - stable API version은 k8s major 버전 내의 모든 향후 release에서 계속 사용할 수 있으며, 현재로서는 stable API를 제거하는 k8s의 major 버전 revision 계획은 없다.

## API groups
[API groups](https://github.com/kubernetes/design-proposals-archive/blob/main/api-machinery/api-group.md)은 k8s API 확장을 쉽게 만들어준다. REST API path와 serialized object의 `.apiVersion`에 API group이 명시된다.

k8s는 몇 가지 API groups이 있다.
- core(또는 legacy) group은 REST path `/api/v1`이고 `apiVersion: v1`로 사용한다. name group 규칙과 동일하게 생각하기 위해 `/api/core/v1`에서 core가 생략된 것이라고 생각하면 된다.
- named group은 REST path `/apis/$GROUP_NAME/$VERSION`이고 `apiVersion: $GROUP_NAME/$VERSION`로 사용한다.

모든 API group은 [Kubernetes API reference](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.30)를 참고한다.

## Enabling or disabling API groups
특정 resource와 API group은 기본적으로 활성화되어 있다. kube-apiserver의 `--runtime-config`를 설정하여 활성화하거나 비활성화할 수 있다. `--runtime-config` flag는 쉼표로 구분된 `<key>[=<value>]` 쌍을 허용한다.` =<value>` 부분이 생략되면 `=true`가 기본값으로 처리된다. 예를 들면 다음과 같다.
- `batch/v1`을 비활성화하려면 `--runtime-config=batch/v1=false`로 설정
- `batch/v2alpha1`을 활성화하려면 `--runtime-config=batch/v2alpha1`로 설정
- `storage.k8s.io/v1beta1/csistoragecapacities`와 같은 특정 버전의 API를 활성화하려면 `--runtime-config=storage.k8s.io/v1beta1/csistoragecapacities`로 설정

> **참고**:
> group이나 resource를 활성화 또는 비활성화할 때 `--runtime-config` 변경 사항을 적용하려면 kube-apiserver, controller-manager를 다시 시작해야한다.

## Persistence
k8s는 serialized state를 etcd에 기록해 저장한다.

