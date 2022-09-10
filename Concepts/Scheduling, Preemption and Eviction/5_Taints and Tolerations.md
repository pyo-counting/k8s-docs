no affinity는 no 집합(요구사항 또는 선호도)을 필터링 및 선택할 수 있는 po의 속성이다. 반면에 taint는 그 반대로 no가 po 집합을 제외할 수 있다.

toleration은 po에 적용된다. toleration을 통해 scheduler는 매칭되는 taint가 있는 po를 스케줄할 수 있다. toleration은 스케줄링을 허용하지만 보장하지는 않는다: scheduler는 그 기능의 일부로서 다른 매개변수를 고려한다.

taint, toleration은 함께 동작하여 po가 부적절한 no에 스케줄되지 않게 한다. 하나 이상의 taint가 no에 적용된다. 이것은 no가 taint를 용인하지 않는 po를 수용해서는 안 되는 것을 나타낸다.

## Concepts
아래와 같이 `kubectl taint` 명령어를 사용해 no에 taint를 추가할 수 있다:

``` bash
kubectl taint nodes node1 key1=value1:NoSchedule
```

node1 no에 taint를 추가한다. taint에는 키=key1, 값=value1, taint effect=NoSchedule 이 있다. 이는 일치하는 toleration이 없으면 po를 node1 에 스케줄링할 수 없음을 의미한다.

아래 명령어를 사용해 taint를 삭제할 수 있다:

``` bash
kubectl taint nodes node1 key1=value1:NoSchedule-
```

po의 spec에 toleration을 명시할 수 있다. 

## Example Use Cases

## Taint based Evictions

## Taint Nodes by Condition