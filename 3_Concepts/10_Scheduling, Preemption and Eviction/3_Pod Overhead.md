no에서 po를 실행할 때 po는 시스템의 리소스를 사용한다. 이 리소스는 po 내부 container를 실행하는 데 필요한 리소스 외 추가적이 리소스다. k8s는 pod overhead를 사용해 container request, limit과 함께 po 인프라에서 소비되는 리소스를 고려한다.

overhead는 RuntimeClass 리소스에 정의되며 해당 [admission](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/#what-are-admission-webhooks) 단계에서 `.spec.overhead` 필드에 overhead 리소스를 추가한다.

po의 overhead는 po를 스케줄링 때 container 리소스 request의 합에 추가로 고려된다. 마찬가지로 kubelet은 po cgroup의 크기를 결정할 때, po의 eviction ranking을 결정할 때 pod overhead를 포함한다.

## Configuring Pod overhead
RuntimeClass 리소스의 `.overhead` 필드를 사용한다.

## Usage example
pod overhead를 사용하기 위해 overhead 필드를 정의하는 RuntimeClass가 필요하다. 아래 RuntimeClass 정의는 virtual machine과 guest OS를 po마다 실행하기 위해 120Mi 메모리를 사용하는 virtualization container runtime(Firecracker virtual machine monitor와 결합된 Kata Container)을 위해 overhead를 사용한다.
``` yaml
# You need to change this example to match the actual runtime name, and per-Pod
# resource overhead, that the container runtime is adding in your cluster.
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: kata-fc
handler: kata-fc
overhead:
  podFixed:
    memory: "120Mi"
    cpu: "250m"
```

kata-fc RuntimeClass handler를 사용하는 workload는 리소스 할당량 계산, no의 스케줄링, po cgroup 크기 조정을 위해 memory, cpu overhead를 고려한다.

위 RuntimeClass를 사용해 test-pod workload를 실행하는 예시를 상상해본다.
``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  runtimeClassName: kata-fc
  containers:
  - name: busybox-ctr
    image: busybox:1.28
    stdin: true
    tty: true
    resources:
      limits:
        cpu: 500m
        memory: 100Mi
  - name: nginx-ctr
    image: nginx
    resources:
      limits:
        cpu: 1500m
        memory: 100Mi
```

RuntimeClass [admission controller](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/)는 RuntimeClass에 정의된 overhead를 포함하도록 workload의 PodSpec을 업데이트한다. PodSpec에 이 필드가 이미 정의되어 있다면 po는 거절된다. 위 예시에서는 RuntimeClass만 지정했기 때문에 admission controller는 po에 overhead를 포함하도록 변경한다.

RuntimeClass admission controller가 수정을 완료한 후, kubectl 명령어를 사용해 overhead 값을 확인할 수 있다.
``` sh
kubectl get pod test-pod -o jsonpath='{.spec.overhead}'
```

위 명령어의 출력 결과는 다음과 같다.
``` sh
map[cpu:250m memory:120Mi]
```


## Verify Pod cgroup limits


### Observability
[kube-state-metrics](https://github.com/kubernetes/kube-state-metrics)에서 제공하는 `kube_pod_overhead_*` metric을 통해 pod overhead에 대한 모니터링을 수행할 수 있다.