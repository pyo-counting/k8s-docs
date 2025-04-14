k8s cluster에서 no는 계획대로 graceful shutdown되거나 예상치 않게 shutdown될 수 있다. shutdown 전에 no가 drain되지 않으면 worklod failure를 유발할 수 있다.

## Graceful node shutdown
kubelet은 node system의 shutdown 감지를 시도하고 실행 중인 po를 종료한다.

kubelet은 node shutdown 동안 일반적인 [pod termination process](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination)를 보장한다. shutdown 중에 kubelet은 새로운 po를 허용하지 않는다(po가 이미 no에 bound되어 있더라도).

gracefule node shutdown은 [systemd inhibitor lock](https://www.freedesktop.org/wiki/Software/systemd/inhibit/)을 이용해 주어진 시간 동안 node의 종료를 지연시키기 때문에 systemd에 의존한다.

graceful node shutdown은 GracefulNodeShutdown feature gate(k8s 1.21부터 기본 활성화)에 의해 제어된다.

기본적으로 `.shutdownGracePeriod`, `.shutdownGracePeriodCriticalPods` 옵션은 값이 0으로 설정되어 gracefule node shutdown 기능을 활성화시키지 않는다. 이 기능을 활성화하기 위해 kubelet에 해당 옵션이 0이 아닌 값으로 변경되어야 한다.

systemd가 no 종료를 감지하게 되면 kubelet은 no의 Ready conditions을 False status로 설정하고 이유를 "node is shutdown"으로 설정한다. kube-scheduler는 이 condition을 존중하며 no에 po를 스케줄링하지 않는다. 다른 third-party scheduler도 동일한 로직을 따를 것으로 예상된다. 이는 해당 no에 새로운 po가 스케줄링되지 않음을 의미한다.

그리고 kubelet은 no의 shutdown이 감지됐을 때 kubelet은 PodAdmission phase의 po도 거부하므로 `node.kubernetes.io/not-ready:NoSchedule` toleration이 있는 po도 거부한다.

graceful shutdown 동안, kubelet은 2개의 phase를 통해 po를 종료한다:
1. no에 실행 중인 regular po 종료
2. no에 실행 중인 [critical po](https://kubernetes.io/docs/tasks/administer-cluster/guaranteed-scheduling-critical-addon-pods/#marking-pod-as-critical) 종료

graceful node shutdown 기능은 2개의 [KubeletConfiguration](https://kubernetes.io/docs/tasks/administer-cluster/kubelet-config-file/) 옵션을 통해 설정된다.
- `shutdownGracePeriod`: no가 shutdown을 지연할 총 시간을 나타낸다. 이는 regular, critical po에 대한 총 po 종료 시간을 나타낸다.
- `shtdownGracePeriodCriticalPods`: critical po의 종료에 사용될 시간을 타나낸다. 해당 옵션은 shutdownGracePeriod보다 작아야 한다.

> **Note**:  
> system에 의해(또는 관리자에 의해) no의 termination이 취소되는 경우도 있다. 이 경우 no는 Ready state로 돌아온다. 하지만 po가 이미 termination을 시작한 경우 kubelet에 의해 다시 복구 될수는 없으며 다시 스케쥴링되어야 한다.

예를 들어 shutdownGracePeriod=30, shtdownGracePeriodCriticalPods=10일 경우, kubelet은 node shutdown을 30초 지연한다. shutdown 동안 20초 (30 - 10)는 normal po를 종료하는 시간으로 예약되며, 이 후 10초는 critical po를 종료하는 시간으로 예약된다.

> **Note**:  
> graceful node shutdown 동안 eviction된 po는 shutdown으로 마킹된다. kubectl get po 명령어를 사용해 eviction된 po가 Terminated 상태임을 확인할 수 있다.
> ```
> Reason:         Terminated
> Message:        Pod was terminated in response to imminent node shutdown.
> ```

### Pod Priority based graceful node shutdown

## Non Graceful node shutdown
사용자 에러(shutdownGracePeriod, ShutdownGracePeriodCriticalPods을 잘못 설정), kubelet이 사용하는 inhibitor locks mechanism이 트리거되지 않아 kubelet의 Node Shutdown Manager가 node의 shutdown 동작을 인지하지 못할 수도 있다.

sts의 po는 shutdown no에서 terminating status에 갇히게 된다. kubelet이 po를 삭제할 수 없기 때문에 sts는 동일한 이름의 po를 새로 생성할 수 없다. po가 사용하는 volume이 있는 경우 shutdown no에서 VolumeAttachments이 삭제되지 않기 때문에 volume을 새로운 no에 사용이 불가하다. 결과적으로 sts에서 실행되는 애플리케이션이 적절하게 기능을 수행할 수 없다. 만약 shutdown 됐던 no가 돌아오면 po는 kubelet에 의해 삭제되고 po는 다른 no에 실행 될 것이다. no가 돌아오지 못하면 해당 po는 terminating status로 평생 남게 된다.

위의 상황을 완화하기 위해 사용자는 no에 수등으로 `taint node.kubernetes.io/out-of-service` taint(NoExecute 또는 NoSchedule) 를 추가해서 no를 out-of-service로 마킹할 수 있다. 만약 kube-controller-manager에 NodeOutOfServiceVolumeDetach feature gate가 활성화 되어있고 no가 위 taint를 갖는 경우 po에 toleration이 없으며 강제로 삭제되고 no에서 종료되는 po에 대한 volume detach 작업이 즉시 수행된다. 이를 통해 다른 no에서 po를 빠르게 다시 실행할 수 있다.

node non-graceful shutdown 동안 po는 두 단계를 통해 삭제된다.
1. out-of-service toleration이 없은 po는 강제 삭제한다.
2. 삭제되는 po에 대한 volume을 즈깃 detach한다.

> **Note**:  
> - `node.kubernetes.io/out-of-service` taint를 추가하기 전에 no가 shutdown 됐는지 확인이 필요하다(재시작 중이면 안됨).
> - The user is required to manually remove the out-of-service taint after the pods are moved to a new node and the user has checked that the shutdown node has been recovered since the user was the one who originally added the taint.

### Forced storage detach on timeout
no가 unhealthy 상태이고 6분 동안 po 삭제에 성공하지 못하면 k8s는 강제로 volume을 detach한다. 해당 no에서 volume을 여전히 사용하는 po는 [CSI specification](https://github.com/container-storage-interface/spec/blob/master/spec.md#controllerunpublishvolume)(ControllerUnpublishVolume. "must be called after all NodeUnstageVolume and NodeUnpublishVolume on the volume are called and succeed")를 위반할 수 있다. 이러한 상황에서 no의 volume은 데이터에 대한 손상을 겪을 수 있다.

force storage detach on timeout는 optional이기 때문에 사용자는 "Non-graceful node shutdown" 기능을 대신 사용할 수도 있다.

force storage detach on timeout은 kube-controller-manager의 disable-force-detach-on-timeout 설정을 사용해 비활성화할 수 있습니다. force detach on timeout 기능을 비활성화하면 6분 이상 작동하지 않는 no에서 호스팅되는 volume에 해당하는 [VolumeAttachment](https://kubernetes.io/docs/reference/kubernetes-api/config-and-storage-resources/volume-attachment-v1/)가 삭제되지 않는다.

이 설정이 적용된 후에도 volume에 연결된 unhealthy po는 위에서 언급한 Non-Graceful Node Shutdown 절차를 통해 복구해야 한다.

> **Note**:  
> - Non-Graceful Node Shutdown 사용에 주의해야 한다.
> - 위 문서화된 단계에서 벗어나면 데이터 손상이 발생할 수도 있다.