API-initiated eviction는 eviction API를 사용해 graceful pod termination을 트리거하는 eviction object를 생성한다.

eviction API를 직접 호출하거나 kubectl drain 명령어를 사용해 eviction을 요청할 수 있다. 그러면 eviction object가 생성되고 kube-apiserver는 po를 종료시킨다.

API-initiated eviction는 po의 `PodDisruptionBudgets`, `terminationGracePeriodSeconds`을 존중한다.

API를 사용해 po에 대한 eviction object를 생성하는 것은 po에 대한 delete 작업을 수행하는 것과 동일하다.

## Calling the Eviction API
k8s POST API를 사용해 eviction object를 생성할 수 있다. 아래는 예시다.
- policy/v1
    ``` json
    {
        "apiVersion": "policy/v1",
        "kind": "Eviction",
        "metadata": {
        "name": "quux",
        "namespace": "default"
        }
    }
    ```

또는 curl, wget과 같은 명령어를 사용할 수 있다.
``` sh
curl -v -H 'Content-type: application/json' https://your-cluster-api-endpoint.example/api/v1/namespaces/default/pods/quux/eviction -d @eviction.json
```

## How API-initiated eviction works
API를 사용해 eviction을 요청하는 경우 kube-apiserver는 admission을 확인 후 아래와 같은 응답을 한다.
- `200 OK`: eviction이 허용되어 object가 생성되고 po가 삭제된다.
- `429 Too Many Requests`: PodDisruptionBudget로 인해 현재 eviction이 허용되지 않는다. 나중에 eviction을 다시 시도해야 한다. 또는 API rate limit으로 인해 발생할 수도 있다.
- `500 Internal Server Error`: 예를 들어 동일 po에 대한 여러 PodDisruptionBudgets 참조과 같은 잘못된 설정이 있을 경우 eviction이 허용되지 않는다.

eviction 대상 po가 PodDisruptionBudget가 있는 workload에 포함되지 않는 경우 kube-apiserver는 항상 200을 응답하고 eviction을 허용한다.

kube-apiserver가 eviction을 허용한 경우 아래 단계를 거쳐 po는 삭제된다.
1. kube-apiserver 내 po는 deletion timestsamp를 추가한다. 그 후 kube-apiserver는 po가 종료된 것으로 간주한다. The Pod resource is also marked with the configured grace period.
2. 해당 po가 실행되는 no의 kubelet은 이를 감지하고 po를 종료시키기 시작한다.
3. kubelet이 po를 shutdown하는 동안, control plane은 ep, endpointslices에서 po를 제거한다. 결과적으로 controller는 po를 유효한 object로 여기지 않는다.
4. po의 terminationGracePeriodSeconds가 경과한 후, kubelet은 po를 강제 종료한다.
5. kubelet은 kube-apiserver에게 po를 삭제하라고 말한다.
6. kube-apiserver는 po를 삭제한다.

## Troubleshooting stuck evictions
경우에 따라 애플리케이션이 broken state로 전환되어 eviction API가 429, 500 HTTP status code만 반환할 수도 있다. 예를 들어 rs가 애플리케이션에 대한 po를 생성했지만 Ready state에 진입하기 전에 발생할 수 있다. 또는 마지막으로 evicted된 po의 terminationGracePeriodSeconds가 클 경우에도 발생할 수 있다.

eviction에서 행이 걸린 경우 아래 해결 방법을 사용할 수 있다.
- 문제의 원인이 되는 자동화된 작업을 중지 또는 일시 중지한다. 작업을 다시 시작하기 전에 행걸린 애플리케이션을 살펴본다.
- 잠시 기다렸다가 eviction API를 사용하는 대신 cluster control plane에서 po를 직접 삭제한다.