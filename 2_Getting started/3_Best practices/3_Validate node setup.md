## Node Conformance Test
node conformance test는 no을 위한 시스템 검증, 기능 테스트를 제공하는 컨테이너 환경의 테스트 프레임워크다. 테스트는 no가 k8s의 최소 요구 사항을 충족하는지 확인한다. 테스트를 통과한 no는 k8s에 join할 자격이 있음을 의미한다.

## Node Prerequisite
node conformance test를 실행하기 위해 no가 표준 k8s no와 동일한 요구 사항을 충족해야 한다. 최소한 아래와 같은 daemon이 설치되어 있어야 한다.
- CRI 호환 container runtime: containerd, CRI-O 등
- kubelet

## Running Node Conformance Test
node conformance test을 실행하기 위해 아래 단계를 수행한다.
1. kubelet의 --kubeconfig 옵션 값을 확인한다. 예를 들어, `--kubeconfig=/var/lib/kubelet/config.yaml`. 테스트 프레임워크는 kubelet을 테스트하기 위해 로컬 환경에 control plane을 실행하기 때문에 kube-apiserver URL로 http://localhost:8080을 사용한다. 사용 가능한 kubelet flag는 다음과 같다.
    - `--cloud-provider`: `--cloud-provider=gce`를 사용하면 해당 flag를 지우고 테스트를 수행한다.
2. 아래 명령어를 통해 node conformance test를 수행한다.
    ``` sh
    # $CONFIG_DIR is the pod manifest path of your Kubelet.
    # $LOG_DIR is the test output path.
    sudo docker run -it --rm --privileged --net=host \
      -v /:/rootfs -v $CONFIG_DIR:$CONFIG_DIR -v $LOG_DIR:/var/result \
      registry.k8s.io/node-test:0.2
    ```
## Running Node Conformance Test for Other Architectures
node conformance test는 cpu 아키텍처별로 docker image를 제공한다.
| Arch  |      Image      |
|-------|:---------------:|
| amd64 | node-test-amd64 |
| arm   |  node-test-arm  |
| arm64 | node-test-arm64 |

## Running Selected Test
특정 테스트를 실행하기 위해 FOCUS 환경 변수에 정규표현식을 사용할 수 있다.
``` sh
sudo docker run -it --rm --privileged --net=host \
  -v /:/rootfs:ro -v $CONFIG_DIR:$CONFIG_DIR -v $LOG_DIR:/var/result \
  -e FOCUS=MirrorPod \ # Only run MirrorPod test
  registry.k8s.io/node-test:0.2
```

특정 테스트를 실행하지 않기 위해 SKIP 환경 변수에 정규표현식을 사용할 수 있다.
``` sh
sudo docker run -it --rm --privileged --net=host \
  -v /:/rootfs:ro -v $CONFIG_DIR:$CONFIG_DIR -v $LOG_DIR:/var/result \
  -e SKIP=MirrorPod \ # Run all conformance tests but skip MirrorPod test
  registry.k8s.io/node-test:0.2
```

node conformance test는 [node e2e test](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-node/e2e-node-tests.md)의 container 버전이다. 기본적으로 모든 테스트를 수행한다.

이론적으로 container를 제대로 구성하고 필요한 볼륨을 mount하면 어떤 노드 e2e든 테스트 가능하다. 그러나 non-conformance test를 실행하는 것은 훨씬 더 복잡한 설정이 필요하므로 conformance test만 실행하는 것을 권장한다.


## Caveats
- 테스트 완료 후 no에 node conformance test image, 기능 테스트를 위해 사용한 테스트 image가 남는다.
- 테스트 완료 후 no에 test에 사용된 죽은 container들이 있다.