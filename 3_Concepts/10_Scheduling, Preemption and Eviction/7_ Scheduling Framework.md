스케줄링 프레임워크는 k8s kube-scheduler를 위한 plugin 아키텍처다. 이는 kube-scheduler에 직접 컴파일된 "plugin" API 집합으로 구성된다. API를 사용해 대부분의 스케줄링 기능을 plugin으로 구현할 수 있으며 동시에 스케줄링 "core"를 가볍고 관리 가능하게 한다. 스케줄링 프레임워크의 설계에 대한 기술적 자세한 내용은 [design proposal of the scheduling framework](https://github.com/kubernetes/enhancements/blob/master/keps/sig-scheduling/624-scheduling-framework/README.md)을 참고한다.

## Framework workflow
스케줄링 프레임워크는 몇 가지 extension point을 정의한다. 스케줄러 plugin은 하나 이상의 extension point에서 호출되도록 등록한다. 이러한 plugin 중 일부는 스케줄링 결정을 변경할 수 있고 일부는 정보 제공에 불과하다.

하나의 po를 스케줄링하기 위한 프로세스는 scheduling cycle, binding cycle로 이루어진다.

### Scheduling cycle & binding cycle
scheduling cycle은 po를 위한 no를 선택하고 binding cycle은 해당 결정을 cluster에 적용한다. 이 두 cycle을 "scheduling context"라고 부른다.

scheduling cycle은 직렬로 실행되고, binding cycle은 병렬로 동시에 실행될 수 있다.

po가 스케줄링 불가하거나 내부 오류가 있는 경우 두 cycle을 중단할 수도 있다. 그러면 po는 queue에 다시 반환되고 재시도 된다.

## Interfaces
아래 그림은 po의 scheduling context와 스케줄링 프레임워크가 노출하는 interface를 보여준다.

하나의 plugin은 더 복잡하고 statefule 작업을 수행하기 위해 여러 interface를 구현할 수도 있다.

일부 interface는 [Scheduler Configuration](https://kubernetes.io/docs/reference/scheduling/config/#extension-points)을 통해 설정할 수 있는 scheduler extension point에 매칭된다.

![](https://kubernetes.io/images/docs/scheduling-framework-extensions.png)