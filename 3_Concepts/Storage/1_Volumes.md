## Background
k8s는 다양한 유형의 volume 타이블 지원한다. po는 여러 volume 타입을 동시에 사용할 수 있다. ephemeral volume 타입은 po와 같은 lifetime을 갖지만 persistent volume은 po의 lifetime과 관계없다. po가 삭제되면 ephemeral volume도 삭제되지만 persistent volume은 삭제되지 않는다.

기본적으로 volume은 디렉터리로 po 내 container에서 접근할 수 있다.

po를 위해 제공 할 volume은 .spec.volumes에 정의하며 container에 volume을 mount하기위해 .spec.containers[*].volumeMounts를 설정한다. 

volume은 다른 volume 내에 마운트될 수 없다. 또한 volume은 다른 volume 내 content에 대한 hard link를 포함할 수 없다.

## Types of Volumes

### configMap
cm은 설정 데이터를 po내 주입(inject)하는 방법을 제공한다. cm에 저장된 데이터는 configMap 타입의 volume에서 참조되고 po내 실행되는 container에서 접근할 수 있다.

**Note**:
- volume 사용 이전에 cm을 먼저 생성해야 한다.
- cm을 subPath volume으로 사용할 때 cm에 대한 업데이트가 반영되지 않는다.
- 텍스트 데이터는 UTF-8 인코딩을 사용한 파일로 저장된다. 다른 문자 인코딩의 경우 binaryData를 이용해야 한다.

### downwardAPI
volume 내에서 노출된 데이터를 일반 텍스트 형식의 읽기 전용 파일로 사용할 수 있다.

**Note**: downward API를 subpath volume으로 사용할 때 업데이트가 반영되지 않는다.

### emptyDir
emptyDir volume은 po가 no에 할당될 때 처음 생성되며, po가 실행되는 동안에만 존재한다. emptyDir volume은 초기에 비어있다.

**Note**: container가 crash 될 때 po는 no에서 삭제되지 않는다. 그렇기 때문에 container crash로부터 emptyDir volume의 데이터는 안전하다.

emptyDir.medium 필드를 "Memory"로 설정하면 k8s는 tmpfs(RAM 기반 파일시스템)를 사용한다.

**Note**: SizeMemoryBackedVolumes feature gate가 활성화되면 memory 기반 volume에 대해 크기를 지정할 수 있다. 크기를 지정하지 않으면 리눅스 호스트 메모리의 50%로 조정된다.

### hostPath
**Warning**: hostPath volume에는 많은 보안 결함이 있기 때문에 가능하면 사용하지 않는 것이 좋다. hostPath volume을 사용해야 하는 경우 ReadOnly로 마운트하는 것을 권장한다.

AdmissionPolicy를 사용해 특정 디렉토리의 hostPath 접근을 제한하는 경우, readOnly 마운트를 사용하는 정책이 유효하기 위해 volumeMounts 필드가 반드시 지정되어야 한다.

### local
local volume은 디스크와 같은 로컬 스토리지 장치를 나타낸다.

local volume은 정적으로 생성된 pv로만 사용할 수 있다. 동적 provision은 지원하지 않는다.

hostPath volume과 비교했을 때 local volume은 수동으로 no에 po를 스케줄링할 필요가 없다. pv는 no의 affinity를 확인해 volume의 no 제약 조건을 확인한다.

그러나 local volume은 여전히 no의 가용성을 따르며 모든 애플리케이션에 적합하지는 않다. 만약 no가 비정상 상태가 되면 local volume도 접근할 수 없게 되고, po를 실행할 수 없게 된다. local volume을 사용하는 애플리케이션은 기본 디스크의 내구 특성에 따라 이러한 감소되는 가용성과 데이터 손실 가능성도 허용할 수 있어야 한다.

아래는 local volume, nodeAffinity를 사용하는 pv의 예시다:

``` yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: example-pv
spec:
  capacity:
    storage: 100Gi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-storage
  local:
    path: /mnt/disks/ssd1
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - example-node
```

local volume을 사용하는 경우 pv nodeAffinity를 설정해야 한다. k8s scheduler는 pv nodeAffinity를 사용하여 po를 올바른 no로 스케줄한다.

pv의 volumeMode을 "Block" (기본값은 "Filesystem")으로 설정하면 local volume을 raw block 장치로 노출할 수 있다.

local volume을 사용할 때는 volumeBindingMod 가 WaitForFirstConsumer로 설정된 StorageClass를 생성하는 것을 권장한다. 자세한 내용은 local [StorageClas](https://kubernetes.io/docs/concepts/storage/storage-classes/#local) 예제를 참고한다. volume 바인딩을 지연시키는 것은 PersistentVolumeClaim 바인딩 결정이 no 리소스 요구사항, node selector, pod affinity, pod anti-affinity와 같이 po가 가질 수 있는 다른 no 제약 조건으로 평가되도록 만든다.

local volume 라이프사이클의 향상된 관리를 위해 외부 정적 provisioner를 별도로 실행할 수 있다. 이 provisioner는 아직 동적 프로비저닝을 지원하지 않는다. 외부 로컬 provisioner를 실행하는 방법에 대한 예시는 [local volume provisioner user guide](https://github.com/kubernetes-sigs/sig-storage-local-static-provisioner) 페이지를 참고한다.

**Note**: 외부 정적 provisioner를 사용해 volume 라이프사이클을 관리하지 않는 경우 local pv을 수동으로 정리하고 삭제하는 것이 필요하다.

### persistentVolumeClaim

### projected
projected volume은 여러 기존 volume을 동일한 디렉터리에 매핑한다.

### secret

### 