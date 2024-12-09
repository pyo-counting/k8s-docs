
## Container images

## Container runtimes
container runtime은 k8s 환경에서 container 실행과 lifecycle을 담당한다.

k8s는 containerd, CRI-O와 같이 [Kubernetes CRI(Container Runtime Interface)](about:blank) 구현한 container runtime을 지원한다.

일반적으로 기본 container runtime을 사용한다. 하지만 여러 container runtime을 사용하는 경우 po의 `.spec.runtimeClassName`을 이용해 cluster에 존재하는 [RuntimeClass](https://kubernetes.io/docs/concepts/containers/runtime-class/) object를 명시해 특정 container runtime을 사용할 수도 있다.