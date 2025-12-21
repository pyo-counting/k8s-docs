## Workload placement
deploy, job과 같은 k8s 표준 workload resource는 pod의 lifecycle을 관리하지만 전체 pod 그룹을 하나의 단위로 처리해야 하는 복잡한 scheduling 요구 사항이 발생할 수 있다.

workload API를 사용하면 pod 그룹을 정의하고 [gang scheduling](https://kubernetes.io/docs/concepts/scheduling-eviction/gang-scheduling/)와 같은 고급 scheduling 정책을 pod 그룹에 적용할 수 있다. 이러한 방법은 "all-or-nothing" 배치가 필수적인 배치 처리 및 머신러닝 워크로드에 특히 유용하다.