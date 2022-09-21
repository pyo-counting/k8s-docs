## Background
k8s는 다양한 유형의 volume 타입 지원한다. po는 여러 volume 타입을 동시에 사용할 수 있다. ephemeral volume 타입은 po와 같은 lifetime을 갖지만 persistent volume은 po의 lifetime과 관계없다. po가 삭제되면 ephemeral volume도 삭제되지만 persistent volume은 삭제되지 않는다.

기본적으로 volume은 디렉터리로 po 내 container에서 접근할 수 있다.

po를 위해 제공할 volume은 .spec.volumes에 정의하며 container에 volume을 mount하기위해 .spec.containers[*].volumeMounts를 설정한다. 

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

local volume을 사용할 때는 volumeBindingMode가 WaitForFirstConsumer로 설정된 StorageClass를 생성하는 것을 권장한다. 자세한 내용은 local [StorageClas](https://kubernetes.io/docs/concepts/storage/storage-classes/#local) 예제를 참고한다. volume 바인딩을 지연시키는 것은 PersistentVolumeClaim 바인딩 결정이 no 리소스 요구사항, node selector, pod affinity, pod anti-affinity와 같이 po가 가질 수 있는 다른 no 제약 조건을 통해 평가되도록 만든다.

local volume 라이프사이클의 향상된 관리를 위해 외부 정적 provisioner를 별도로 실행할 수 있다. 이 provisioner는 아직 동적 프로비저닝을 지원하지 않는다. 외부 로컬 provisioner를 실행하는 방법에 대한 예시는 [local volume provisioner user guide](https://github.com/kubernetes-sigs/sig-storage-local-static-provisioner) 페이지를 참고한다.

**Note**: 외부 정적 provisioner를 사용해 volume 라이프사이클을 관리하지 않는 경우 local pv을 수동으로 정리하고 삭제하는 것이 필요하다.

### persistentVolumeClaim
pvc volume은 po에 pv를 마운트하기 위해 사용된다. pvc를 사용해 사용자는 특정 cloud 환경에 대한 상세 사항을 알 필요 없이 스토리지를 "요청"할 수 있다.
### projected
projected volume은 여러 기존 volume을 동일한 디렉터리에 매핑한다.

### secret
secret volume은 po에 패스워드와 같이 민감한 정보를 전달하는 데 사용된다. k8s API에 secret을 생성하고 po에 파일로 노출시킬 수 있다. secret volume은 tmpfs(RAM 지원 파일시스템)를 사용한다.

**NOte**: volume으로 사용하기 전에 secret을 먼저 생성해야 한다.

**Note**: -secret을 subPath volume으로 사용할 때 cm에 대한 업데이트가 반영되지 않는다.

## Using subPath
volumeMounts.subPath을 사용해 volume 내 특정 경로만 container에 마운트 할 수 있다.

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-lamp-site
spec:
    containers:
    - name: mysql
      image: mysql
      env:
      - name: MYSQL_ROOT_PASSWORD
        value: "rootpasswd"
      volumeMounts:
      - mountPath: /var/lib/mysql
        name: site-data
        subPath: mysql
    - name: php
      image: php:7.0-apache
      volumeMounts:
      - mountPath: /var/www/html
        name: site-data
        subPath: html
    volumes:
    - name: site-data
      persistentVolumeClaim:
        claimName: my-lamp-site-data
```

### Using subPath with expanded environment variables
subPathExpr 필드를 사용해 downward API 환경 변수로부터 subPath 디렉토리 이름을 설정한다. subPath와 subPathExpr은 같이 사용할 수 없다.

아래 예시에서 po는 subPathExpr을 사용하여 hostPath volume /var/log/pods 디렉토리 내 pod1 디렉토리를 생성한다. hostPath volume은 downward API에서 po 이름을 가져온다. 호스트 디렉토리 /var/log/pods/pod1은 컨테이너의 /logs에 마운트된다.

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod1
spec:
  containers:
  - name: container1
    env:
    - name: POD_NAME
      valueFrom:
        fieldRef:
          apiVersion: v1
          fieldPath: metadata.name
    image: busybox:1.28
    command: [ "sh", "-c", "while [ true ]; do echo 'Hello'; sleep 10; done | tee -a /logs/hello.txt" ]
    volumeMounts:
    - name: workdir1
      mountPath: /logs
      # The variable expansion uses round brackets (not curly brackets).
      subPathExpr: $(POD_NAME)
  restartPolicy: Never
  volumes:
  - name: workdir1
    hostPath:
      path: /var/log/pods
```

## Resources 
emptyDir volume의 저장 매체는 kubelet 루트 디렉토리(일반적으로 /var/lib/kubelet)의 파일시스템 매체에 의해 결정된다. emptyDir 또는 hostPath volume이 소비할 수 있는 크기에 대한 제한이 없으며 container 또는 po간 격리도 없다.

## Out-of-tree volume plugins

## Mount propagation
mount propagation은 동일 po내 container 사이, 또는 동일 no에 다른 po간 volume을 공유할 수 있도록 한다.

mount propagation은 container.volumeMounts의 mountPropagation 필드를 통해 제어된다.

- None: 
- HostToContainer:
- Bidirectional:

### Configuration
일부 배포(CoreOS, RedHat/Centos, Ubuntu)에서 mount propagation가 제대로 동작하기 위해 아래와 같이 docker 내 mount share를 올바르게 설정해야 한다.

docker systemd service 파일에 아래 내용을 설정한다.

```
MountFlags=shared
```

또는 MountFlags=slave가 존재하는 경우 삭제한다. 그리도 docker 데몬을 재시작한다.