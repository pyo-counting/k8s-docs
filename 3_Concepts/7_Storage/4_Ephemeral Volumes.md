ephemeral volume은 po의 라이프사이클과 동일하다. 그렇기 때문에 pv의 이용 가능성 여부와 상관없이 po는 중지, 실행될 수 있다. ephemeral volume는 po의 .spec에서 inline으로 정의된다.

## Types of ephemeral volumes
k8s는 다양한 목적을 위해 여러 ephemeral volume을 지원한다:

- emptyDir: po가 시작될 때 빈 상태로 시작되며 로컬 kubelet base 디렉토리(보통 루트 디스크) 또는 메모리로 제공된다.
- configMap, downwardAPI, secret: k8s의 각 resource의 데이터를 po에 주입한다.
- CSI ephemeral volume: 위 설명된 volume 들과 비슷하지만. 이를 지원하는 CSI 드라이버에 의해서만 제공될 수 있다.
- generic ephemeral volume: pv를 지원하는 모든 스토리지 드라이버가 제공할 수 있다(po의 .spec.volumes[*]에서 pvc를 명시해서 사용).

emptyDir, configMap, downwardAPI, secret은 [local ephemeral storage](https://v1-23.docs.kubernetes.io/docs/concepts/configuration/manage-resources-containers/#local-ephemeral-storage)로 제공된다. 이들은 각 no의 kubelet에 의해 관리된다.

CSI ephemeral volume은 third-party CSI storage driver로 제공된다.

generic ephemeral volume은 third-party CSI storage driver로 제공될 수 있지만 dynamic provisioning을 지원하는 다른 storage driver에 의해 제공될 수도 있다. 일부 CSI driver는 CSI ephmeral volume을 위해서만 설계됐을 수도 있기 때문에 generic ephemeral volume으로 사용할 수 없다.

third-party driver를 사용하면 k8s가 제공하지 않는 기능을 사용할 수 있다.

## CSI ephemeral volumes
**Note**: CSI ephemeral volume은 일부 CSI driver에서만 지원된다. 관련 목록은 [Drivers list](https://kubernetes-csi.github.io/docs/drivers.html) 페이지에서 확인한다.

개념적으로 CSI ephemeral volume은 configMap, downwardAPI, secret volume 타입과 유사하다: 스토리지는 각 no에서 관리되며 no에 po가 스케쥴링된 후 다른 로컬 리소스들과 같이 생성된다. k8s는 po에 대한 재스케쥴링 개념이 없다. 그렇기 때문에 volume 생성에 대한 실패 가능성이 낮아야 한다. 그렇지 않으면 po의 실행 단계에서 중지된다. 특히 스토리지 용량을 고려한 po 스케쥴링을 지원하지 않는다. 또한 kubelet이 자체 관리하는 스토리지에만 용량 리소스 제한을 할 수 있기 때문에 po의 스토리지 리소스 사용 제한이 적용되지 않는다.

아래는 CSI ephemeral volume을 사용하는 po 예시다:

``` yaml
kind: Pod
apiVersion: v1
metadata:
  name: my-csi-app
spec:
  containers:
    - name: my-frontend
      image: busybox:1.28
      volumeMounts:
      - mountPath: "/data"
        name: my-csi-inline-vol
      command: [ "sleep", "1000000" ]
  volumes:
    - name: my-csi-inline-vol
      csi:
        driver: inline.storage.kubernetes.io
        volumeAttributes:
          foo: bar
```

.spec.volumes[*].csi.volumeAttributes 필드를 사용해 driver가 준비할 volume을 명시한다. 이러한 attribute는 각 driver에 따라 다르며 표준화되지 않았다. 

## CSI driver restrictions
CSI ephemeral volume을 위해 po의 .spec.volumes[*].csi.volumeAttributes 필드를 사용한다. 일반적으로 관리자가 해당 필드를 관리하도록 허용하는 CSI driver의 경우 inline ephemeral volume을 사용하기에는 적합하지 않다. 예를 들어, sc에 정의되는 파라미터는 inline ephemeral volume을 통해 사용자에게 노출되면 안된다.

- CSIDriver spec의 volumeLifecycleModes에서 Ephemeral을 삭제함으로써 inline ephemeral volume으로 사용되는 것을 막는다.
- admission webhook을 사용해 driver의 사용에 대한 제한을한다.

## Generic ephemeral volumes
generic ephemeral volume은 프로비저닝 후 일반적으로 비어 있는 디렉토리를 각 po에 제공한다는 점에서 emptyDir volume과 유사하다. 하지만 다음과 같은 추가 기능도 제공한다.

- 스토리지는 로컬이거나 네트워크 연결형(network-attached)일 수 있다.
- volume의 크기를 고정할 수 있으며 po는 이 크기를 초과할 수 없다.
- 드라이버와 파라미터에 따라 volume이 초기 데이터를 가질 수도 있다.
- volume에 대한 일반적인 작업은 드라이버가 지원하는 범위 내에서 지원된다. 이와 같은 작업은 다음을 포함한다: snapshotting, cloning, resizing, storage capacity tracking.

예시:

``` yaml
kind: Pod
apiVersion: v1
metadata:
  name: my-app
spec:
  containers:
    - name: my-frontend
      image: busybox:1.28
      volumeMounts:
      - mountPath: "/scratch"
        name: scratch-volume
      command: [ "sleep", "1000000" ]
  volumes:
    - name: scratch-volume
      ephemeral:
        volumeClaimTemplate:
          metadata:
            labels:
              type: my-frontend-volume
          spec:
            accessModes: [ "ReadWriteOnce" ]
            storageClassName: "scratch-storage-class"
            resources:
              requests:
                storage: 1Gi
```

## Lifecycle and PersistentVolumeClaim
volume claim을 위한 파라미터를 po의 spec.volume에서 정의할 수 있다. label, annotation, pvc을 위한 모든 필드가 지원된다. po가 생성되면, ephemeral volume controller는 po가 속한 동일한 ns에 pvc objetc를 생성하고 po가 삭제될 때에는 pvc도 삭제되도록 만든다.

sc의 volumeBindingMode 필드가 Immediate 또는 WaitForFirstConsumer 인지에 따라 volume binding과 provisioning을 트리거한다. generic ephemeral volume의 경우 후자를 권장하는데, 이 경우 scheduler가 po를 할당하기에 적합한 no를 선택하기가 쉬워지기 때문이다. VolumeBindingImmediate을 사용하는 경우 scheduler는 volume이 사용 가능해지는 즉시 해당 volume에 접근 가능한 no를 선택하도록 강요받는다.

resource ownwership과 관련해 generic ephemeral volume을 갖는 po는 generic ephemeral volume을 제공하는 pvc의 소유자이다. po가 삭제되면, k8s gc는 해당 pvc를 삭제하는데 sc의 기본 reclaim 정책이 volume을 삭제하는 것이기 때문에 pvc의 삭제는 보통 volume의 삭제를 유발한다. reclaim 정책을 retain으로 설정한 sc를 사용해 준 임시(quasi-ephemeral) 로컬 스토리지를 생성할 수도 있는데, 이렇게 하면 스토리지의 수명이 po의 수명보다 길어지며 이러한 경우 volume 정리를 별도로 수행해야 함을 명심해야 한다.

이러한 pvc가 존재하는 동안은 다른 pvc와 동일하게 사용될 수 있다. 특히, volume 복제 또는 스냅샷 시에 데이터 소스로 참조될 수 있다. 또한 해당 pvc object는 해당 volume의 현재 상태도 가지고 있다.

## PersistentVolumeClaim naming
자동으로 생성된 pvc의 이름은 규칙에 따라 정해진다: pvc의 이름은 po 이름과 volume 이름의 사이를 하이픈(-)으로 결합한 형태다. 위의 예시에서, pvc 이름은 my-app-scratch-volume가 된다. 이렇게 규칙에 의해 정해진 이름은 pvc와의 상호작용을 더 쉽게 만드는데, 이는 po 이름과 volume 이름을 알면 pvc 이름을 별도로 검색할 필요가 없기 때문이다.

pvc 명명 규칙에 따라 서로 다른 po 간 이름 충돌이 발생할 수 있으며("pod-a" po + "scratch" volume vs. "pod" po + "a-scratch" volume - 두 경우 모두 pvc 이름은 "pod-a-scratch") 또한 po와 수동으로 생성한 pvc 간에도 이름 충돌이 발생할 수 있다.

이러한 충돌은 감지될 수 있다: pvc는 po를 위해 생성된 경우에만  ephemeral volume를 위해 사용될 수 있다. 이러한 검사는 소유권 관계를 기반으로 한다. 기존에 존재하던 pvc는 덮어써지거나 수정되지 않는다. 물론 충돌을 해결해주지는 않는데 이는 적합한 pvc가 없이는 po가 시작될 수 없기 때문이다.

**Caution**: 동일 ns에서 po, volume의 이름이 충돌하지 않도록 유의해야 한다.

## Security
GenericEphemeralVolume 기능을 활성화하면 사용자가 po를 생성할 수 있는 경우 pvc를 간접적으로 생성할 수 있도록 허용하며, 심지어 사용자가 pvc를 직접적으로 만들 수 있는 권한이 없는 경우에도 이를 허용한다. 클러스터 관리자는 이를 명심해야 한다. 이것이 보안 모델에 부합하지 않는다면, 다음의 두 가지 선택지가 있다.

- generic ephemeral volume을 갖는 po와 같은 object를 거부하는 admission webhook을 사용한다.
- volume의 목록 중에 ephemeral volume 타입을 포함하지 않도록 psp를 사용한다(k8s 1.21에서 사용 중단됨).

일반적인 pvc의 네임스페이스 쿼터는 여전히 적용되므로, 사용자가 이 새로운 메카니즘을 사용할 수 있도록 허용되었어도, 다른 정책을 우회하는 데에는 사용할 수 없다.