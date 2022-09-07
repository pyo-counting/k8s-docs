ns ojbect에 label을 추가해 pod security standard를 설정할 수 있다. privileged, baseline, restricted 3 가지 정책은 보안 스펙트럼을 광범위하게 다루며 po seuciry admission controller에 의해 구현된다.

## Before you begin

## Requiring the baseline Pod Security Standard with namespace labels
아래 manifest는 my-baseline-namespace 이름을 갖는 ns를 정의한다:

- baseline 정책 요구 사항을 만족하지 않는 po는 거절한다.
- restricted 정책 요구 사항을 만족하지 않는 po가 생성되면 사용자에게 경고를 노출하고 audit annotation을 추가한다.
- baseline, restricted 정책의 버전을 v1.25로 설정한다.

``` yaml
apiVersion: v1
kind: Namespace
metadata:
  name: my-baseline-namespace
  labels:
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/enforce-version: v1.25

    # We are setting these to our _desired_ `enforce` level.
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: v1.25
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: v1.25
```

## Add labels to existing namespaces with kubectl label
**Note**: enforce 모드에 대한 label이 추가 또는 변경되면, admission 플러그인은 새로운 정책에 대해 현재 ns에 있는 모든 po를 검사한다. 그리고 위반에 대한 경고가 사용자에게 노출된다.

--dry-run 플래그를 사용해 ns에 대한 보안 profile 변경 사항을 미리 확인하는 것이 좋다. pod security standard는 dry run mode에서 실행되어 실제 정책을 업데이트 하지는 않으며 새 정책이 기존 po를 처리하는 것에 대한 정보를 제공할 뿐이다.

``` bash
kubectl label --dry-run=server --overwrite ns --all \
    pod-security.kubernetes.io/enforce=baseline
```

## Applying to all namespaces

## Applying to a single namespace