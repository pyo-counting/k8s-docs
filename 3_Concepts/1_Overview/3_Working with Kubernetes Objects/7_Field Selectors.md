k8s의 resource 필드 값에 대한 field selecotr를 사용해 resource를 필터링 할 수 있다. kubectl get 명렁어 기준 —field-selector flag를 사용하면 된다.

``` bash
kubectl get pods --field-selector status.phase=Running
```

## Supported fields
k8s resource 타입에 따라 지원 가능한 필드가 다양하다. 기본적으로 모든 resource 타입에 대해 `metadata.name`, `metadata.namespace` 필드를 지원한다. 

## Supported operators
=, ==, != 연산자를 지원한다.

## Chained selectors

## Multiple resource types