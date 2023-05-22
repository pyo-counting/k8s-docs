container 내 디스크에 저장된 파일은 수명이 짧다(ephemeral). container의 state는 저장되지 않기 때문에 container의 lifetime 간 생성, 수정된 파일은 저장되지 않는다. crash가 발생하면 kubelet은 clean state로 container를 재시작한다. 또 다른 문제는 po 내 container간 파일을 공유다. 이러한 문제를 해결하기 위해 k8s volume이라는 추상적 개념이 존재한다.

## Background
k8s는 다양한 유형의 volume 타입 지원한다. po는 여러 volume 타입을 동시에 사용할 수 있다. ephemeral volume 타입은 po와 같은 lifetime을 갖지만 persistent volume은 po의 lifetime과 관계없다. po가 삭제되면 ephemeral volume도 삭제되지만 persistent volume은 삭제되지 않는다.

volume의 핵심은 데이터를 포함하는 디렉토리로써 po 내 container에서 접근할 수 있다는 것이다. How that directory comes to be, the medium that backs it, and the contents of it are determined by the particular volume type used.

po를 위해 사용할 volume은 .spec.volumes에 정의하며 container에 volume을 mount하기위해 .spec.containers[*].volumeMounts를 설정한다. container 내 프로세스는 container image의 내용과 container에 마운트된 volume으로 구성된 파일시스템 뷰를 본다. volume은 image 내 지정된 경로에 마운트된다. po 내에 정의된 각 container마다 container가 사용하는 volume의 마운트 위치를 각각 지정해야 한다.

volume은 다른 volume 내에 마운트될 수 없다. 또한 volume은 다른 volume 내 content에 대한 hard link를 포함할 수 없다.

## Types of Volumes
k8s는 몇 가지 종류의 volume 타입을 지원한다.

### awsElasticBlockStore (deprecated)

### azureDisk (deprecated)

### azureFile (deprecated)

### cephfs

### cinder (deprecated)

### configMap
cm은 설정과 관련된 데이터를 po내 주입(inject)하는 방법을 제공한다. cm에 저장된 데이터는 configMap 타입의 volume에서 참조되고 po내 실행되는 container에서 접근할 수 있다.

아래는 예시다:

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-pod
spec:
  containers:
    - name: test
      image: busybox:1.28
      volumeMounts:
        - name: config-vol
          mountPath: /etc/config
  volumes:
    - name: config-vol
      configMap:
        name: log-config
        items:
          - key: log_level
            path: log_level
```

위에서 log-config cm의 log_level key를 갖는 데이터가 test container 내 /etc/config 위치에 config-vol 파일로 마운트된다.

**Note**:
- volume 사용 이전에 cm을 먼저 생성해야 한다.
- cm을 subPath volume으로 사용할 때 cm에 대한 업데이트가 반영되지 않는다.
- 텍스트 데이터는 UTF-8 인코딩을 사용한 파일로 저장된다. 다른 문자 인코딩의 경우 binaryData를 이용해야 한다.

### downwardAPI
downwardAPI volume을 사용하면 downwardAPI 데이터를 애플리케이션에서 사용할 수 있다. volume에 노출된 데이터를 일반 텍스트 형식의 읽기 전용 파일로 사용할 수 있다.

**Note**: downward API를 subpath volume으로 사용할 때 업데이트가 반영되지 않는다.

### emptyDir
emptyDir volume은 po가 no에 할당될 때 처음 생성되며, po가 실행되는 동안에만 존재한다. emptyDir volume은 초기에 비어있다. po의 모든 container는 emptyDir volume을 마운트해 해당 volume에 존재하는 파일에 대해 모두 읽기/쓰기가 가능하다. po가 해당 no에서 삭제된다면 데이터 역시 모두 삭제된다.

**Note**: container가 crash 될 때 po는 no에서 삭제되지 않는다. 그렇기 때문에 container crash로부터 emptyDir volume의 데이터는 안전하다.

emptyDir.medium 필드를 통해 emptyDir volume이 저장될 위치를 지정한다. 기본적으로 no의 환경에 따라 디스크, SSD, 네트워크 스토리지 등이 될 수 있다. emptyDir.medium 필드를 "Memory"로 설정하면 k8s는 tmpfs(RAM 기반 파일시스템)를 사용한다. tmpfs는 매우 빠르지만 디스크와 달리 no 재부팅 시 지워지고 쓰기 파일은 container memory limit에 영향을 받는다.

기본 medium에 대해서는 크기를 제한해 emptyDir volume의 용량을 제한할 수 있다. 만약 다른 소스(예를 들어 로그 파일)에 의해 용량이 채워지면 emptyDir의 크기 제한 이전에 용량이 부족해질 수도 있다.

**Note**: SizeMemoryBackedVolumes feature gate가 활성화되면 memory 기반 volume에 대해 크기를 지정할 수 있다. 크기를 지정하지 않으면 리눅스 호스트 메모리의 50%로 조정된다.

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec:
  containers:
  - image: registry.k8s.io/test-webserver
    name: test-container
    volumeMounts:
    - mountPath: /cache
      name: cache-volume
  volumes:
  - name: cache-volume
    emptyDir:
      sizeLimit: 500Mi
```

### fc (fibre channel)

### gcePersistentDisk (deprecated)

### gitRepo (deprecated)

### glusterfs (removed)

### hostPath
**Warning**: hostPath volume에는 많은 보안 결함이 있기 때문에 가능하면 사용하지 않는 것이 좋다. hostPath volume을 사용해야 하는 경우 ReadOnly로 마운트하는 것을 권장한다.

AdmissionPolicy를 사용해 특정 디렉토리의 hostPath 접근을 제한하는 경우, readOnly 마운트를 사용하는 정책이 유효하기 위해 volumeMounts 필드가 반드시 지정되어야 한다.

hostPath volume은 no의 파일시스템에 존재하는 파일 또는 디렉토리를 po에 마운트한다. 대부분의 po에서는 필요하지 않지만 일부 애플리케이션에 대해 powerful escape hatch를 제공할 수 있다.

아래는 예시다:

- Docker에 대한 접근이 필요한 경우 no의 /var/lib/docker를 마운트
- cAdvisor를 실행하기 위해 no의 /sys를 마운트

path 필드외 type 필드를 지정할 수 있다. type 필드에 대해 가능한 값은 아래와 같다:

| Value             | Behavior                                                                                                                                                               |   |   |   |
|-------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---|---|---|
|                   | Empty string (default) is for backward compatibility, which means that no checks will be performed before mounting the hostPath volume.                                |   |   |   |
| DirectoryOrCreate | If nothing exists at the given path, an empty directory will be created there as needed with permission set to 0755, having the same group and ownership with Kubelet. |   |   |   |
| Directory         | A directory must exist at the given path                                                                                                                               |   |   |   |
| FileOrCreate      | If nothing exists at the given path, an empty file will be created there as needed with permission set to 0644, having the same group and ownership with Kubelet.      |   |   |   |
| File              | A file must exist at the given path                                                                                                                                    |   |   |   |
| Socket            | A UNIX socket must exist at the given path                                                                                                                             |   |   |   |
| CharDevice        | A character device must exist at the given path                                                                                                                        |   |   |   |
| BlockDevice       | A block device must exist at the given path      

#### hostPath configuration example
``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec:
  containers:
  - image: registry.k8s.io/test-webserver
    name: test-container
    volumeMounts:
    - mountPath: /test-pd
      name: test-volume
  volumes:
  - name: test-volume
    hostPath:
      # directory location on host
      path: /data
      # this field is optional
      type: Directory
```

**Caution**: The FileOrCreate mode does not create the parent directory of the file. If the parent directory of the mounted file does not exist, the pod fails to start. To ensure that this mode works, you can try to mount directories and files separately, as shown in the FileOrCreateconfiguration.

#### hostPath FileOrCreate configuration example
``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-webserver
spec:
  containers:
  - name: test-webserver
    image: registry.k8s.io/test-webserver:latest
    volumeMounts:
    - mountPath: /var/local/aaa
      name: mydir
    - mountPath: /var/local/aaa/1.txt
      name: myfile
  volumes:
  - name: mydir
    hostPath:
      # Ensure the file directory is created.
      path: /var/local/aaa
      type: DirectoryOrCreate
  - name: myfile
    hostPath:
      path: /var/local/aaa/1.txt
      type: FileOrCreate
```

### iscsi

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

local volume을 사용할 때는 volumeBindingMode가 WaitForFirstConsumer로 설정된 StorageClass를 생성하는 것을 권장한다. 자세한 내용은 local [StorageClas](https://kubernetes.io/docs/concepts/storage/storage-classes/#local) 예제를 참고한다. volume 바인딩을 지연시키는 것은 no 리소스 요구사항, node selector, pod affinity, pod anti-affinity와 같이 po가 가질 수 있는 no 제약 조건이 pvc 바인딩 결정에 영향을 줄 수 있음을 의미한다.

local volume 라이프사이클의 향상된 관리를 위해 외부 정적 provisioner를 별도로 실행할 수 있다. 이 provisioner는 아직 동적 프로비저닝을 지원하지 않는다. 외부 로컬 provisioner를 실행하는 방법에 대한 예시는 [local volume provisioner user guide](https://github.com/kubernetes-sigs/sig-storage-local-static-provisioner) 페이지를 참고한다.

**Note**: 외부 정적 provisioner를 사용해 volume 라이프사이클을 관리하지 않는 경우 local pv을 수동으로 정리하고 삭제하는 것이 필요하다.

### nfs

### persistentVolumeClaim
pvc volume은 po에 pv를 마운트하기 위해 사용된다. pvc를 사용해 사용자는 특정 cloud 환경에 대한 상세 사항을 알 필요 없이 스토리지를 "요청"할 수 있다.

### portworxVolume (deprecated)

### projected
projected volume은 여러 기존 volume을 동일한 디렉터리에 매핑한다.

### rbd

### secret
secret volume은 po에 패스워드와 같이 민감한 정보를 전달하는 데 사용된다. k8s API에 secret을 생성하고 po에 파일로 노출시킬 수 있다. secret volume은 tmpfs(RAM 지원 파일시스템)를 사용한다.

**Note**: volume으로 사용하기 전에 secret을 먼저 생성해야 한다.

**Note**: secret을 subPath volume으로 사용할 때 secret에 대한 업데이트가 반영되지 않는다.

### vsphereVolume (deprecated)

## Using subPath
.spec.containers[*].volumeMounts[*].subPath을 사용해 volume 내 특정 경로만 container에 마운트 할 수 있다.

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

아래 예시에서 po는 subPathExpr을 사용하여 hostPath volume /var/log/pods 디렉토리 내 pod1 디렉토리를 가리킨다. hostPath volume은 downward API에서 po 이름을 가져온다. 호스트 디렉토리 /var/log/pods/pod1은 컨테이너의 /logs 디렉토리 내에 마운트된다.

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
out-of-tree volume plugin은 CSI(Container Storage Interface), FlexVolume(deprecated)를 포함한다. 이러한 플러그인을 통해 storage vendor가 k8s 저장소에 플러그인 소스코드를 추가하지 않고도 커스텀 storage plugin을 생성할 수 있다.

이전의 모든 volume plugin은 "in-tree"였다. "in-tree" plugin은 코어 k8s 바이너리와 함께 빌드, 링크, 컴파일 됐었다. 즉, 새로운 storage 시스템을 k8s에 추가하기 위해서는 코어 k8s 코드 저장소의 코드를 확인해야 했다는 것이다.

CSI, FlexVolume을 사용하면 k8s 코드와 상관없이 volume plugin을 개발 및 k8s cluster extension으로 배포할 수 있다.

### csi
Container Storage Interface(CSI)는 Container Orchestration system(COs)(예를 들어 k8s)의 container workload에 arbitary storage를 제공하기 위한 표준 인터페이스를 정의한다.

**Note**: Support for CSI spec versions 0.2 and 0.3 are deprecated in Kubernetes v1.13 and will be removed in a future release.

**Note**: CSI drivers may not be compatible across all Kubernetes releases. Please check the specific CSI driver's documentation for supported deployments steps for each Kubernetes release and a compatibility matrix.

CSI와 호한되는 volume driver가 k8s 클러스터에 배포되면 사용자는 csi volume type을 사용해 CSI driver를 통해 노출된 volume을 마운트할 수 있다.

csi volume은 po에서 3가지 방식을 통해 사용 가능하다:

- pvc
- generic ephemeral volume
- driver가 지원하는 CSI ephemeral volume 

CSI persistent volume을 설정하기 위해 stroage 관리자는 아래 필드를 사용할 수 있다:

- driver:
- volumeHandle:
- readOnly:
- fsType:
- volumeAttributes:
- controllerPublishSecretRef
- CSINodeExpandSecret feature gate:
- nodePublishSecretRef:
- nodeStageSecretRef:

#### CSI raw block volume support
외부 CSI driver를 사용하는 벤더사는 k8s 워크로드에서 raw block volume 지원을 구현할 수 있다.

특정 CSI에 대한 변화 없이 일반적일 방식을 통해 설정 가능하다. 관련된 내용은 [PersistentVolume/PersistentVolumeClaim with raw block volume support](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#raw-block-volume-support) 페이지를 참고한다.

#### CSI ephemeral volumes
po의 .spec에서 직접 CSI volume을 설정할 수 있다. 이러한 방식으로 명시되는 volume을 ephemeral volume이라고 칭하며 po의 라이프사이클을 따른다. 

#### Migrating to CSI drivers from in-tree plugins
CSIMigration 기능은 기존 in-tree 플러그인에 대한 작업을 관련 CSI plugin(설치 및 설정 필요)으로 연결시킨다. 결과적으로 관리자는 in-tree 플러그인을 대체하는 CSI driver로 전환할 떄 기존 sc, pv, pvc를 따로 설정할 필요가 없다.

지원되는 기능은 다음과 같다: provisioning/delete, attach/detach, mount/unmount and resizing of volumes.

CSIMigration을 지원하고 해당 CSI driver를 구현하는 in-tree 플러그인 볼륨 타입은 [Types of Volumes](https://kubernetes.io/docs/concepts/storage/volumes/#volume-types) 페이지를 참고한다.

### flexVolume (deprecated)

## Mount propagation
mount propagation은 동일 po내 container 사이, 또는 동일 no에 다른 po간 volume을 공유할 수 있도록 한다.

mount propagation은 container.volumeMounts의 mountPropagation 필드를 통해 제어된다. 해당 필드에 사용 가능한 값은 다음과 같다:

- None: volume mount는 해당 volume에 대한 후속 마운트, 호스트에 의한 하위 디렉토리에 대한 것을 수신하지 않는다. 비슷한 방식으로 container에 의해 생성된 마운트는 호스트에서 보이지 않는다. 이는 기본 mode다.

  이 mode는 [mount(8)](https://man7.org/linux/man-pages/man8/mount.8.html)에 설명된 rprivate mount propagation과 동일하다.

  하지만 rprivate propagation이 적용 불가한 경우 CRI runtime은 대신 rslave mount propagation을 선택한다. cri-dockerd(Docker)는 mount source가 Docker daemon의 root directory(기본적으로 /var/llib/docker)를 포함하는 경우 rslave mount propagation을 선택하는 것으로 알려져있다.

- HostToContainer: volume mount는 해당 volume, 하위 디렉터리에 마운트된 모든 후속 마운트를 수신한다.

  즉, 호스트가 volume mount에 추가적으로 마운트를 수행하더라도 container는 해당 정보를 볼 수 있다.

  마찬가지로, 동일한 volume에 대한 Bidirectional mount propagation이 설정된 po가 해당 volume에 추가적인 마운트를 수행하면 HostToContainer mount propagation 설정의 container가 해당 정보를 볼 수 있다.

  이 mode는 [mount(8)](https://man7.org/linux/man-pages/man8/mount.8.html)에 설명된 rslave mount propagation과 동일하다.
- Bidirectional: volume mount는 HostToContainer mount와 동일하게 동작한다. 추가적으로 container에 의해 생성된 volume mount도 host, 동일한 volume을 사용하는 모든 po의 container로도 전파된다.

  해당 mode에 대한 전형적인 사용 예는 FlexVolume, CSI driver po 또는 hostpath volume를 사용하는 po다.

  이 mode는 [mount(8)](https://man7.org/linux/man-pages/man8/mount.8.html)에 설명된 rshared mount propagation과 동일하다.

**Warning**: Bidirectional mount propagation can be dangerous. It can damage the host operating system and therefore it is allowed only in privileged containers. Familiarity with Linux kernel behavior is strongly recommended. In addition, any volume mounts created by containers in pods must be destroyed (unmounted) by the containers on termination.

### Configuration
일부 배포(CoreOS, RedHat/Centos, Ubuntu)에서 mount propagation가 제대로 동작하기 위해 아래와 같이 docker 내 mount share를 올바르게 설정해야 한다.

docker systemd service 파일에 아래 내용을 설정한다.

```
MountFlags=shared
```

또는 MountFlags=slave가 존재하는 경우 삭제한다. 그리도 docker 데몬을 재시작한다.