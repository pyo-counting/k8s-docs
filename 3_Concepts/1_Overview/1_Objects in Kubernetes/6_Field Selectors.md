k8s의 resource 필드 값에 대한 field selecotr를 사용해 resource를 필터링 할 수 있다. kubectl get 명렁어 기준 --field-selector flag를 사용하면 된다.

``` bash
kubectl get pods --field-selector status.phase=Running
```

> **Note**:  
> field selector는 resource 필터다. 기본적으로 selector/filter가 적용되지 않으므로 지정된 타입의 모든 리소스가 선택된다. 즉 아래 두 명령어는 동일하다.
> `kubectl get pods`  
> `kubectl get pods --field-selector ""`

## Supported fields
k8s resource 타입에 따라 지원 가능한 필드가 다양하다. 기본적으로 모든 resource 타입에 대해 `metadata.name`, `metadata.namespace` 필드를 지원한다. 

## Supported operators
=, ==, != 연산자를 지원한다. =, ==은 동일한 의미다.

> **Note**:  
> label selector와 같이 [Set-based operators](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/#set-based-requirement)는 제공하지 않는다.

## Chained selectors
label selector와 같이 콤마 문자(,)를 사용해 여러 field selector를 사용할 수 있다.

## Multiple resource types
