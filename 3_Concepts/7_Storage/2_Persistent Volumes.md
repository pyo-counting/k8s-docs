스토리지 관리는 컴퓨트 인스턴스 관리와는 별개의 문제다. PersistentVolume 서브시스템은 사용자 및 관리자에게 스토리지 사용 방법에서부터 스토리지가 제공되는 방법에 대한 세부 사항을 추상화하는 API를 제공한다. 이를 위해 pv, pvc이라는 두 가지 새로운 API 리소스를 소개한다(po의 .spec.volumes에서는 GCE persistent disk, AWS Elastic Block Store을 직접 참조해 사용할 수 있다).

pv(non-namespaced)는 관리자가 직접 프로비저닝하거나 sc를 사용하여 동적으로 프로비저닝한 클러스터의 스토리지이다. no가 클러스터 resource인 것처럼 pv는 클러스터 resource이다. pv는 Volumes와 같은 volume plguin이지만 pv를 사용하는 개별 po와는 별개의 라이프사이클을 가진다. 이 API object는 NFS, iSCSI 또는 cloud-provider-specific storage system 등 스토리지 구현에 대한 세부 정보를 담아낸다.

pvc(namespaced)은 사용자의 스토리지에 대한 요청이다. po와 비슷하다. po는 노드 리소스를 사용하고 pvc는 pv resource를 사용한다. po는 특정 수준의 리소스(CPU 및 메모리)를 요청할 수 있다. 클레임은 특정 크기 및 접근 모드를 요청할 수 있다(예: ReadWriteOnce, ReadOnlyMany 또는 ReadWriteMany로 마운트 할 수 있음. AccessModes 참고).

pvc을 사용하면 사용자가 추상화된 스토리지 리소스를 사용할 수 있지만 사용자는 요구 사항에 따라 각기 다른 pv가 필요한 경우가 일반적이다. 클러스터 관리자는 사용자에게 해당 volume의 구현 방법에 대한 세부 정보를 제공하지 않고 크기와 접근 모드 뿐만 아니라 여러 특성을 고려한 pvc을 제공할 수 있어야 한다. 이러한 요구 사항을 위해 sc 리소스가 있다.

## Lifecycle of a volume and claim
pv는 non-namespaced resource다. pvc는 pv에 대한 요청(claim)이며 resource에 대한 클레임 검사 역할을 한다. pv와 pvc 간의 상호 작용은 다음 라이프사이클을 따른다.

### Provisioning
pv는 두 가지 방식을 통해 프로비저닝 된다: static, dynamic

#### Static
클러스터 관리자가 pv를 직접 생성한다. pv는 실제 스토리지에 대한 상세 사항을 저장하고 있으며 클러스터 사용자가 사용할 수 있다.

#### Dynamic
관리자가 생성한 static pv가 사용자의 pvc과 매칭되지 않으면 클러스터는 volume을 동적 프로비저닝을 시도한다. 이 프로비저닝은 sc를 기반으로 한다. pvc는 sc를 요청해야 하며 관리자는 동적 프로비저닝이 동작할 수 있도록 sc를 관리해야 한다. pvc 내 sc 필드가 ""라면 pvc는 동적 프로비저닝을 비활성화한다.

sc를 기반으로 동적 스토리지 프로비저닝을 사용하려면 클러스터 관리자가 API server내 DefaultStorageClass admission controller를 활성화해야 한다. API server의 --enable-admission-plugins flag 값 목록중 DefaultStorageClass가 포함되어 있는지 확인 및 설정하면 된다.

### Binding
master 내 control loop는 새로운 pvc를 감시하고 요청(claim)과 매칭되는 pv가 있을 경우 pvc와 pv를 binding한다. 새로운 pvc로부터 pv가 dynamic provision된다면 loop는 pv와 pvc를 항상 binding할 수 있다. Otherwise, the user will always get at least what they asked for, but the volume may be in excess of what was requested. 일단 binding이 되면 방식에 관계 없이 pvc binding은 베타적이다. pvc - pv binding은 양방향 binding인 ClaimRef(pv의 .spec.claimRef 필드)를 사용하는 1:1 매핑이다.

매칭되는 volume이 없는 경우 요청은 unbounded 상태로 유지된다. 매칭되는 volume이 존재하게 될 경우 pvc는 pv와 binding된다. 예를 들어, 클러스터에 50Gi의 pv가 많을 때, 100Gi를 요청하는 pvc는 클러스터에 100Gi pv가 추가될 때 까지 unbounded 상태로 남아있는다.

### Using
po는 volume을 사용하기 위해 pvc를 사용한다. 클러스터는 요청을 확인해 binding된 volume을 찾아 해당 volume을 po에 마운트한다. 여러 접근 모드를 지원하는 pv의 경우 사용자는 po에서 해당 volume을 어떤 모드로 사용할지 pvc를 통해 명시한다.

사용자는 po의 .spec.volumes[*] 필드에 pvc를 사용해 요청한 pv에 접근할 수 있는 po를 생성할 수 있다.

### Storage Object in Use Protection
Storage Object in Use Protection의 기능은 pvc와 pvc에 binding된 pv가 시스템에서 삭제되지 않도록 하기 위한 것이다. 즉, 데이터 유실을 막기 위해서다.

**Note**:  PVC is in active use by a Pod when a Pod object exists that is using the PVC.

사용자가 po에 의해 사용 중인 pvc를 삭제할 경우, pvc는 바로 삭제되지 않는다. po가 pvc를 더 이상 사용하지 않을 때까지 pvc에 대한 삭제는 미뤄진다. 또한 pvc에 binding된 pv를 삭제하더라도 pv 역시 바로 삭제되지 않는다. pv에 binding된 pv가 존재하지 않을 때까지 pv에 대한 삭제는 미뤄진다.

pvc의 상태가 Terminating이지만 Finalizer 목록에 kubernetes.io/pvc-protection 이 포함되어 pvc가 보호되는 것을 볼 수 있다:

``` bash
kubectl describe pvc hostpath
Name:          hostpath
Namespace:     default
StorageClass:  example-hostpath
Status:        Terminating
Volume:
Labels:        <none>
Annotations:   volume.beta.kubernetes.io/storage-class=example-hostpath
               volume.beta.kubernetes.io/storage-provisioner=example.com/hostpath
Finalizers:    [kubernetes.io/pvc-protection]
...
```

pv의 상태가 상태가 Terminating이지만 Finalizer 목록에 kubernetes.io/pv-protection 이 포함되어 pv가 보호되는 것을 볼 수 있다.:

``` bash
kubectl describe pv task-pv-volume
Name:            task-pv-volume
Labels:          type=local
Annotations:     <none>
Finalizers:      [kubernetes.io/pv-protection]
StorageClass:    standard
Status:          Terminating
Claim:
Reclaim Policy:  Delete
Access Modes:    RWO
Capacity:        1Gi
Message:
Source:
    Type:          HostPath (bare host directory volume)
    Path:          /tmp/data
    HostPathType:
Events:            <none>
```

### Reclaiming
volume을 다 사용했으면 pvc를 삭제함으로써 리소스에 대한 요청을 회수할 수 있다. pv의 reclaim policy(.spec.persistentVolumeReclaimPolicy 필드)에 따라 release 이후 cluster가 volume을 어떻게 처리할지 결정한다. 현재 volume은 Retained, Recycled, Deleted 될 수 있다.

static pv의 기본 값은 Retain, dynamic pv의 기본 값은 Delete이다.

#### Retain
Retain reclaim policy는 리소스에 대한 수동 회수를 의미한다. pvc가 삭제되면 pv는 삭제되지 않으며 released 상태로 남겨진다. volume에는 이전 데이터가 존재하기 때문에 다른 pvc에서 사용할 수는 없다. 관리자가 직접 아래 단계를 통해 회수를 진행해야 한다:

1. pv를 삭제한다. pv가 삭제되더라도 외부 인프라에 존재하는 storage 자원은 존재한다.
2. storage 자원에 저장된 데이터를 직접 삭제한다.
3. storage 자원을 직접 삭제한다.

동일 storage 자원을 재사용하길 원할 경우 동일 storage 자원 정의를 사용해 pv를 생성한다.

#### Delete
Delete reclaim policy을 지원하는 volume plugin의 경우 k8s의 pv, 외부 storage 자원을 모두 삭제한다. sc에 의해 동적 provision된 volume은 sc의 설정(기본값이 Delete)을 사용한다. The administrator should configure the StorageClass according to users' expectations; otherwise, the PV must be edited or patched after it is created. See Change the Reclaim Policy of a PersistentVolume.

#### Recycle
**Warning**: Recycle reclaim policy는 deprecated다. 대신 동적 provision을 사용하는 것을 권장한다.

Recycle reclaim policy을 지원하는 volume plugin의 경우 `rm -rf /thevolume/*` 명령어를 사용해 volume 내 데이터를 삭제한다. 그렇기 때문에 새로운 pvc에서 사용이 가능하다.

물론 관리자가 k8s controller manager 명령어 인자를 사용해 사용자가 커스텀 recycler를 설정할 수 있다. 커스텀 recycler po template은 아래 예시와 같이 반드시 .spec.volumes[*] 필드를 가지고 있어야 한다:


``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: pv-recycler
  namespace: default
spec:
  restartPolicy: Never
  volumes:
  - name: vol
    hostPath:
      path: /any/path/it/will/be/replaced
  containers:
  - name: pv-recycler
    image: "registry.k8s.io/busybox"
    command: ["/bin/sh", "-c", "test -e /scrub && rm -rf /scrub/..?* /scrub/.[!.]* /scrub/*  && test -z \"$(ls -A /scrub)\" || exit 1"]
    volumeMounts:
    - name: vol
      mountPath: /scrub
```

However, the particular path specified in the custom recycler Pod template in the volumes part is replaced with the particular path of the volume that is being recycled.

### PersistentVolume deletion protection finalizer
Delete reclaim policy를 갖는 pv에 대해 stroage 자원이 삭제된 이후에만 삭제될 수 있도록 finalizer를 추가할 수 있다.

kubernetes.io/pv-controller, external-provisioner.volume.kubernetes.io/finalizer finalizer는 동적으로 provision된 volume에만 추가된다.

kubernetes.io/pv-controller finalizer는 in-tree plugin에 추가된다. 아래는 예시다:

```
kubectl describe pv pvc-74a498d6-3929-47e8-8c02-078c1ece4d78
Name:            pvc-74a498d6-3929-47e8-8c02-078c1ece4d78
Labels:          <none>
Annotations:     kubernetes.io/createdby: vsphere-volume-dynamic-provisioner
                 pv.kubernetes.io/bound-by-controller: yes
                 pv.kubernetes.io/provisioned-by: kubernetes.io/vsphere-volume
Finalizers:      [kubernetes.io/pv-protection kubernetes.io/pv-controller]
StorageClass:    vcp-sc
Status:          Bound
Claim:           default/vcp-pvc-1
Reclaim Policy:  Delete
Access Modes:    RWO
VolumeMode:      Filesystem
Capacity:        1Gi
Node Affinity:   <none>
Message:         
Source:
    Type:               vSphereVolume (a Persistent Disk resource in vSphere)
    VolumePath:         [vsanDatastore] d49c4a62-166f-ce12-c464-020077ba5d46/kubernetes-dynamic-pvc-74a498d6-3929-47e8-8c02-078c1ece4d78.vmdk
    FSType:             ext4
    StoragePolicyName:  vSAN Default Storage Policy
Events:                 <none>
```

external-provisioner.volume.kubernetes.io/finalizer finalizer는 CSI volume에 추가된다 아래는 예시다:

```
Name:            pvc-2f0bab97-85a8-4552-8044-eb8be45cf48d
Labels:          <none>
Annotations:     pv.kubernetes.io/provisioned-by: csi.vsphere.vmware.com
Finalizers:      [kubernetes.io/pv-protection external-provisioner.volume.kubernetes.io/finalizer]
StorageClass:    fast
Status:          Bound
Claim:           demo-app/nginx-logs
Reclaim Policy:  Delete
Access Modes:    RWO
VolumeMode:      Filesystem
Capacity:        200Mi
Node Affinity:   <none>
Message:         
Source:
    Type:              CSI (a Container Storage Interface (CSI) volume source)
    Driver:            csi.vsphere.vmware.com
    FSType:            ext4
    VolumeHandle:      44830fa8-79b4-406b-8b58-621ba25353fd
    ReadOnly:          false
    VolumeAttributes:      storage.kubernetes.io/csiProvisionerIdentity=1648442357185-8081-csi.vsphere.vmware.com
                           type=vSphere CNS Block Volume
Events:                <none>
```

특정 in-tree plugin에 대해 CSIMigration{provider} feature flag가 활성화된 경우 kubernetes.io/pv-controller는 external-provisioner.volume.kubernetes.io/finalizer로 대체된다.

### Reserving a PersistentVolume
pvc를 특정 pv에 바안딩할 수도 있다.

pvc에 pv를 명시함으로써 특정 pv를 pvc를 바인딩할 수 있다. pv에 .spec.claimRef를 사용하지 않은 경우 pvc와 pv는 바인딩 될 수 있다.

바인딩은 node affinity를 포함한 일부 volume 매칭 규칙과 관계없이 수행된다. 물론 control plane은 sc, access mode, 요청 스토리지 크기는 확인한다.

``` yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: foo-pvc
  namespace: foo
spec:
  storageClassName: "" # Empty string must be explicitly set otherwise default StorageClass will be set
  volumeName: foo-pv
  ...
```

해당 방법이 pv에 대한 바인딩을 완전히 보장하지는 않는다. 다른 pvc가 pv를 사용할 수도 있기 때문에 먼저 pv의 .spec.claimRef를 통해 관련 pvc를 명시해야 한다.

``` yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: foo-pv
spec:
  storageClassName: ""
  claimRef:
    name: foo-pvc
    namespace: foo
  ...
```

.spec.persistentVolumeReclaimPolicy을 Retain으로 설정한 pv를 재사용하는 경우에 유용하다.

### Expanding Persistent Volumes Claims
pvc에 대한 용량 확장 기능은 기본적으로 활성화된다. 아래 volume 타입의 경우 확장이 가능하다:

- azureDisk
- azureFile
- awsElasticBlockStore
- cinder (deprecated)
- csi
- flexVolume (deprecated)
- gcePersistentDisk
- rbd
- portworxVolume

.allowVolumeExpansion 필드가 true로 설정된 sc에 대한 pvc에서만 확장을 할 수 있다.

``` yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: example-vol-default
provisioner: vendor-name.example/magicstorage
parameters:
  resturl: "http://192.168.10.100:8080"
  restuser: ""
  secretNamespace: ""
  secretName: ""
allowVolumeExpansion: true
```

pvc를 통해 volume의 크기를 확장하는 경우 pvc 오브젝트를 수정하면된다. 이를 통해 pv에 대한 실제 volume에 대한 용량 확장을 트리거한다. 해당 요청을 위해 새로운 pv가 생성되는 것은 아니며 대신 기존 volume의 크기가 조정된다.

**Warning**: pv 오브젝트를 직접 수정하면 volume에 대한 크기가 자동으로 조정되지 않을 수도 있다. pv의 용량을 먼저 편집한 후 바인딩된 pvc의 용량을 pv의 용량과 동일하게 수정하는 경우에도 storage의 크기는 자동으로 조정되지 않는다. k8s control plane은 두 resource의 desired state를 확인하고 실제 물리 volume 크기가 수동으로 증가했다고 생각해 크기 조정을 수행할 필요가 없다고 결론 짓는다.

#### CSI Volume expansion
CSI volume에 대한 확장은 기본적으로 활성화 되어있지만, CSI driver가 해당 기능을 지원해야 한다. 관련해서는 CSI driver의 문서를 참고한다.

#### Resizing a volume containing a file system 
XFS, Ext3, Ext4 파일시스템을 포함하는 volume에 대해서만 확장이 가능하다.

volume에 파일시스템이 포함된 경우, 새로운 po가 ReadWirte 모드로 설정한 pvc를 사용하는 경우에만 파일시스템이 확장된다. po가 초기화 중(start up)이거나 파일시스템이 온라인 확장을 지원하는 경우에는 po가 실행 중일 때 파일시스템이 확장된다.

FlexVolumes (deprecated since Kubernetes v1.23) allow resize if the driver is configured with the RequiresFSResize capability to true. The FlexVolume can be resized on Pod restart.

#### Resizing an in-use PersistentVolumeClaim
pvc을 사용하는 po를 삭제하고 다시 만들 필요는 없다. 사용 중인 모든 pvc는 파일시스템이 확장되는 즉시 관련 po에서 사용할 수 있다.이 기능은 사용 중이지 않은 pvc에는 영향을 주지 않는다. 확장을 완료하기 위해 먼저 pvc를 사용하는 po를 생성해야 한다.

Similar to other volume types - FlexVolume volumes can also be expanded when in-use by a Pod.

**Note:**: FlexVolume resize is possible only when the underlying driver supports resize.

**Note:**: Expanding EBS volumes is a time-consuming operation. Also, there is a per-volume quota of one modification every 6 hours.

#### Recovering from Failure when Expanding Volumes
실제 stroage system에서 확장되기에는 너무 큰 크기를 사용자가 명시할 경우, pvc는 사용자가 어떠한 조치를 쥐하기 전까지 계속 확장을 재시도한다. 이는 이상적이지 않기 때문에 k8s에서는 이러한 실패상황에 대한 회복을 위해 다음과 같은 방법을 제공한다.

- Manually with Cluster Administrator access: 확장을 실패할 경우, 관리자는 직접 해당 요청을 취소해 pvc의 상태를 원복할 수 있다.

    1. pvc에 바운드된 pv를 Reain reclaim policy로 수정한다.
    2. pvc를 삭제한다. pv는 Reatin reclaim policy이기 때문에 volume은 삭제되지 않는다.
    3. pv의 .spec.claimRef를 삭제함으로써 새로운 pvc가 해당 pv에 바인딩될 수 있도록 한다. 이 작업을 수행하면    pv는 Released -> Available 상태가 된다.
    4. pv보다 작은 크기의 pvc를 다시 생성한다. .spec.volumeName 필드에 pv의 이름을 설정한다.
    5. pv의 reclaim policy를 원래 값으로 다시 수정한다.

- By requesting expansion to smaller size:

    **Note**: 사용자가 pvc 확장 실패로부터의 복구하는 방법은 k8s 1.23버전에서 추가되었으며 alpha 기능이다. RecoverVolumeExpansionFailure 관련 feature gate 기능이 활성화되어야 한다. 

    만약 RecoverVolumeExpansionFailure feature gate가 클러스터 내에서 활성화됐을 경우, 이전 요청 값보다 작은 크기르 확장을 재시도할 수 있다. 이를 위해 pvc의 .spec.resource 필드 값을 더 작게 설정한다. 그리고 .status.resizeStatus, k8s의 events 오브젝트를 확인해 확장에 대한 재시도 상태를 확인할 수 있다. 

    volume에 대한 확장은 지원하지 않기 때문에 처음 volume보다 더 큰 크기로 수정을 해야한다는 것에 주의해야 한다.

## Types of Persistent Volumes
pv는 플러그인으로 구현 및 제공된다. k8s는 현재 아래 목록에 대해 지원한다:

- cephfs - CephFS volume
- csi - Container Storage Interface (CSI)
- fc - Fibre Channel (FC) storage
- hostPath - HostPath volume (for single node testing only; WILL NOT WORK in a multi-node cluster; consider using local volume instead)
- iscsi - iSCSI (SCSI over IP) storage
- local - local storage devices mounted on nodes.
- nfs - Network File System (NFS) storage
- rbd - Rados Block Device (RBD) volume

아래 pv는 deprecated다. 즉, 아직 지원은 하지만 추후 릴리즈 버전에서 삭제될 수 있다:

- awsElasticBlockStore - AWS Elastic Block Store (EBS) (deprecated in v1.17)
- azureDisk - Azure Disk (deprecated in v1.19)
- azureFile - Azure File (deprecated in v1.21)
- cinder - Cinder (OpenStack block storage) (deprecated in v1.18)
- flexVolume - FlexVolume (deprecated in v1.23)
- gcePersistentDisk - GCE Persistent Disk (deprecated in v1.17)
- portworxVolume - Portworx volume (deprecated in v1.25)
- vsphereVolume - vSphere VMDK volume (deprecated in v1.19)

k8s의 이전 버전은 아래 in-tree pv 플러그인을 지원한다:

- photonPersistentDisk - Photon controller persistent disk. (not available starting v1.15)
- caleIO - ScaleIO volume (not available starting v1.21)
- locker - Flocker storage (not available starting v1.25)
- uobyte - Quobyte volume (not available starting v1.25)
- torageos - StorageOS volume (not available starting v1.25)

## Persistent Volumes
모든 pv는 .spec, .statue 필드를 갖는다. pv의 이름은 DNS subdomain name을 준수해야 한다.

``` yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv0003
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: slow
  mountOptions:
    - hard
    - nfsvers=4.1
  nfs:
    path: /tmp
    server: 172.17.0.2
```

**Note**: 클러스터 내에서 pv을 사용하기 위해 pv의 타입과 관련된 헬퍼 프로그램이 필요할 수도 있다. 위 예시에서는 pv가 NFS 타입이기 때문에 NFS 파일시스템에 마운트를 지원하는 헬퍼 프러그램인 /sbin/mount.nfs가 필수적으로 필요하다.

### Capacity
일반적으로 pv는 용량 제한을 갖는다. 이는 .spec.capacity 필드를 사용해 지정한다. 

현재 용량 크기는 설정 및 요청이 가능한 유일한 리소스다. 추후에는 IOPS, 처리량과 같은 리소스가 추가될 수 있다.

### Volume Mode
k8s는 volume mode를 .spec.volumeMode 필드를 통해 지원한다: 사용 가능한 값은 Block, Filesystem이 있다.

해당 필드는 필수가 아니며 기본 값은 Filesystem이다.

Block 값을 사용해 volume을 raw block device로 사용할 수도 있다. 이러한 volume은 po 내에서 파일시스템이 없는 block device로 존재한다. 이는 po와 volume 사이에 파일시스템 계층이 없이 volume에 대한 가장 빠른 접근을 제공하기 위해 유용하다. 다른 한편으로 po 내 애플리케이션은 raw block device를 다루는 방법에 대해 반드시 알아야 한다는 것이다.

### Access Modes
pv는 리소스 provider가 지원하는 방법에 따라 호스트에 마운트될 수 있다. 아래 표와 같이 provider는 서로 다른 기능을 지원할 수 있으며 각각의 pv에 대한 access mode는 특정 volume에서 지원하는 특정 모드로 설정된다. 예를 들어 NFS는 다중 read/write 클라이언트를 지원할 수 있지만 특정 NFS pv는 서버에서 읽기 전용으로 설정될 수 있다. 각 pv는 해당 pv에 대한 고유한 access mode를 갖는다.

.spec.accessModes[*] 필드에 사용 가능한 값은 다음과 같다:

- ReadWriteOnce: volume이 단일 no에 read-write 가능하도록 마운트될 수 있다. ReadWriteOnce access mode는 여전히 po가 실행될 동안 동일 no의 여러 po가 접근할 수 있도록 허용한다.
- ReadOnlyMany: volume이 여러 no에서 read-only 가능하도록 마운트 될 수 있다.
- ReadWriteMany: volume이 여러 no에서 read-write 가능하도록 마운트 될 수 있다.
- ReadWriteOncePod: volume이 단일 po에서 read-write 가능하도록 마운트 될 수 있다. 전체 클러스터에서 1개의 po만 해당 pvc을 읽거나 쓰길 원할 경우 사용한다. 이는 오직 CSI volume, k8s 1.22 이상 버전에서만 지원한다.

[Introducing Single Pod Access Mode for PersistentVolumes](https://kubernetes.io/blog/2021/09/13/read-write-once-pod-access-mode-alpha/) 페이지에서 더 자세한 내용을 설명한다.

CLI에서 accessmode는 아래와 같은 축약어로 표현된다:

- RWO: ReadWriteOnce
- ROX: ReadOnlyMany
- RWX: ReadWriteMany
- RWOP: ReadWriteOncePod

**Note**: k8s는 pvc와 pv를 매칭하는데 volume access mode를 사용한다. 경우에 따라 volume access mode를 사용해 pv가 마운트되는 위치를 제한할 수 있다. volume access mode는 storage가 마운트 된 후에는 write protection을 적용하지 않는다. access mode가 ReadWriteOnce, ReadOnlyMany, ReadWriteMany로 설정됐더라도 volume에 아무런 제약 조건을 설정하지 않는다. 예를 들어 pv가 ReadOnlyMany로 생성되더라도 read-only volume이 된다고 보장되지 않는다. ReadWriteOncePod access mode로 지정된 경우 volume이 제한되며 단일 po에서만 마운트할 수 있다.

**Important**: volume이 여러 access mode를 지원한다고 해도 한 번에 한 개의 access mode를 사용해서 마운트된다. 예를 들어 GCEPersistentDisk는 ReadWriteOnce, ReadOnlyMany mode를 지원하지만 동시에 사용될 순 없다.

| Volume Plugin        |     ReadWriteOnce     |      ReadOnlyMany     |              ReadWriteMany              | ReadWriteOncePod      |
|----------------------|:---------------------:|:---------------------:|:---------------------------------------:|-----------------------|
| AWSElasticBlockStore |           ✓           |           -           |                    -                    | -                     |
| AzureFile            |           ✓           |           ✓           |                    ✓                    | -                     |
| AzureDisk            |           ✓           |           -           |                    -                    | -                     |
| CephFS               |           ✓           |           ✓           |                    ✓                    | -                     |
| Cinder               |           ✓           |           -           | (if multi-attach volumes are available) | -                     |
| CSI                  | depends on the driver | depends on the driver |          depends on the driver          | depends on the driver |
| FC                   |           ✓           |           ✓           |                    -                    | -                     |
| FlexVolume           |           ✓           |           ✓           |          depends on the driver          | -                     |
| GCEPersistentDisk    |           ✓           |           ✓           |                    -                    | -                     |
| Glusterfs            |           ✓           |           ✓           |                    ✓                    | -                     |
| HostPath             |           ✓           |           -           |                    -                    | -                     |
| iSCSI                |           ✓           |           ✓           |                    -                    | -                     |
| NFS                  |           ✓           |           ✓           |                    ✓                    | -                     |
| RBD                  |           ✓           |           ✓           |                    -                    | -                     |
| VsphereVolume        |           ✓           |           -           |    - (works when Pods are collocated)   | -                     |
| PortworxVolume       |           ✓           |           -           |                    ✓                    | -                     |

### Class
pv에는 .spec.storageClassName 필드를 설정할 수 있다. 특정 클래스의 pv는 해당 클래스를 요청하는 pvc에만 바인딩될 수 있다. .spec.storageClassName이 없는 pv는 클래스가 없으며 특정 클래스를 요청하지 않는 pvc에만 바인딩될 수 있다.

과거에는 .spec.storageClassName 필드 대신 volume.beta.kubernetes.io/storage-class annotation을 사용했다. 이 annotation은 여전히 ​​동작하지만 향후 k8s 릴리즈에서 완전히 deprecated 될 예정이다.

### Reclaim Policy
사용 가능한 reclaim policy는 다음과 같다:

- Retain: 수동 reclaim을 지원
- Delete: AWS EBS, GCE PD, Azure Disk, OpenStack Cinder과 같은 실제 storage 자원으 삭제
- Recycle: pv 재활용. 기본 삭제 명령어는 `rm -rf /thevolume/*`

현재 NFS, HostPath만 Recycle을 지원한다. 그리고 AWS EBS, GCE PD, Azure Disk, Cinder만 Recycle을 지원한다.

### Mount Options
k8s 관리자는 pv가 no에 마운트 될 때 추가적인 마운트 옵션을 설정할 수 있다.

**Note**: 모든 pv 타입이 마운트 옵션을 지원하는 것은 아니다.

아래 volume 타입에 대해 .spec.mountOptions[*] 필드를 지원한다:

- awsElasticBlockStore
- azureDisk
- azureFile
- cephfs
- cinder (deprecated in v1.18)
- gcePersistentDisk
- iscsi
- nfs
- rbd
- vsphereVolume

Mount options are not validated. If a mount option is invalid, the mount fails.

과거에는 해당 필드 대신 volume.beta.kubernetes.io/mount-options annotation을 사용했다. 이 annotation은 아직 동작하지만 이후 k8s 릴리즈에서 삭제될 수 있다.

### Node Affinity
**Note**: 대부분의 volume 유형에서 이 필드를 설정할 필요가 없다. AWS EBS, GCE PD, Azure Disk volume block 유형에 대해 자동으로 할당된다. 하지만 local volume의 경우 명시적으로 설정해야 한다.

pv는 no affinity를 지정하여 volume에 접근할 수 있는 no를 제한하는 제약 조건을 정의할 수 있다. pv를 사용하는 po는 no affinity에 의해 선택된 no에만 스케줄링된다. no affinity를 지정하려면 pv의 .spec.nodeAffinity 필드를 이용한다.

### Phase
volume은 아래 phase를 갖는다:

- Available: claim에 대해 아직 바운드 되지 않은 리소스
- Bounded: claim에 바운딩된 volume
- Released: claim이 삭제되었지만, 리소스는 아직 cluster가 reclaim하지 않음
- Failed: volume이 dynamic reclaim에 대해 실패함

CLI를 통해 pvc에 바인딩도니 pv를 확인할 수 있다.

## PersistentVolumeClaims
모든 pvc는 .spec, .statue 필드를 갖는다. pvc의 이름은 DNS subdomain name을 준수해야 한다.

``` yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myclaim
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 8Gi
  storageClassName: slow
  selector:
    matchLabels:
      release: "stable"
    matchExpressions:
      - {key: environment, operator: In, values: [dev]}
```

### Access Modes
pv와 동일

### Volume Modes
pv와 동일

### Resources
po와 같은 claim은 특정한 크기의 리소스를 요청할 수 있다. [resource model](https://git.k8s.io/design-proposals-archive/scheduling/resources.md)은 volume, claim에 공통적으로 적용된다.

### Selectors
.spec.selector 필드를 통해 volume을 필터링할 수 있다. volume의 label이 selector와 매칭하는 경우에만 claim에 바운딩될 수 있다. selector는 두 가지 필드로 구성될 수 있다:

- matchLabels: volume이 동일한 label key, value를 가져야 한다.
- matchExpressions: key, value 목록, 연산자를 사용해 명시된 필수 조건. 유효한 연산자는 In, Not In, Exists, Does Not Exist가 있다.

matchLabels, matchExpressions의 모든 필수 조건은 AND 연산자로 연산된다.

### Class
pvc에는 .spec.storageClassName 필드를 설정할 수 있다. 요청된 클래스의 pv(pvc와 동일한 storageClassName을 가진 pv)만 pvc에 바인드될 수 있다.

pvc는 반드시 클래스를 가질 필요는 없다. .spec.storageClassName이 ""로 설정된 pvc는 항상 클래스가 없는 pvc를 요청하는 것으로 해석되므로 클래스가 없는 pv에만 바인딩될 수 있다(annotation이 없거나 ""로 설정). .spec.storageClassName 필드 자체가 생략된 pvc와는 다르며 DefaultStorageClass admission 플러그인 활성화 여부에 따라 클러스터에서 다르게 처리된다.

- admission 플러그인이 활성화 됐다면 관리자는 기본 sc를 설정한다. .spec.storageClassName 필드가 없는 모든 pvc는 기본 pv에 바인딩 될 수 있다. 기본 sc는 storageclass.kubernetes.io/is-default-class annotation을 true로 생성하면 된다. 관리자가 기본 sc를 설정하지 않는다면 admission 플러그인이 비활성화된 것처럼 동작한다. 1개 이상의 기본 sc가 있다면 admission 플러그인은 모든 pvc에 대한 생성을 금지한다.
- admission 플러그인이 비활성화 됐다면 기본 sc의 개념이 없는 것이다. storageClassName이 ""로 설정된 pvc는 storageClassName이 ""로 설정된 pv에만 바인딩 될 수 있다. 그러나 기본 sc를 사용할 수 있게 되면 storageClassName이 누락된 pvc를 업데이트할 수 있다. pvc가 업데이트되면 더이상 storageClassName이 ""인 pv로 바인딩 되지 않는다.

자세한 내용은 [retroactive default StorageClass assignment](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#retroactive-default-storageclass-assignment) 페이지를 참고한다.

설치 방식에 따라 기본 addon 매니저가 기본 sc를 k8s 클러스터에 배포할 수도 있다.

sc를 요청하는 것 뿐만 아니라 .spec.selector를 명시할 때 두 조건은 모두 AND 연산자로 계산된다.

**Note**: 현재 .spec.selector가 있는 pvc는 dynamic provisioned pv를 가질 수 없다.

과거에서는 volume.beta.kubernetes.io/storage-class annotation가 .spec.storageClassName 필드 대신 사용됐다. 해당 annotation은 아직 사용하 수 있지만 추후에 없어질 수도 있다.

#### Retroactive default StorageClass assignment
클러스터 내 기본 sc가 없을 떄 새 pvc에 대해 .spec.storageClassName 필드 값을 지정하지 않았을 때 기본 sc가 이용 가능할 때까지 pvc는 해당 필드를 설정하지 않은 상태로 남겨진다.

기본 sc가 이용 가능해지면 control plane은 .spec.storageClassName 필드가 없는 pvc를 찾아낸다. 해당 필드가 아예 지정되지 않았거나 값이 빈 경우 control plane은 기본 sc를 매칭시킨다. .spec.stroageClassName 필드가 ""인 pvc가 있을 때 기본 sc를 설정하면 해당 pvc는 업데이트 되지 않는다.

.spec.storageClassName이 ""인 pv에 바인딩을 유지하기 위해 연결된 pvc의 .spec.storageClassName도 ""로 설정해야 한다.

이러한 동작은 관리자가 이전 sc를 먼저 제거한 다음 다른 sc를 만들거나 설정해 기본 sc를 변경하는 데 도움이 된다. 기본 값이 없는 기간동안 생성된 .spec.storageClassName 필드가 없는 pvc에 대해 기본 값이 설정되지 않지만 retroactive 기본 sc 할당으로 인해 기본 값을 변경하는 방법은 안전하다.

## Claims As Volumes
po는 claim을 volume으로 사용함으로 써 storage에 접근한다. claim은 po와 동일한 ns에 존재해야 한다. cluster는 po가 속한 ns에서 claim을 찾고 관련 pv를 찾는다. 그리고나서 volume은 host, po에 마운트된다.

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
    - name: myfrontend
      image: nginx
      volumeMounts:
      - mountPath: "/var/www/html"
        name: mypd
  volumes:
    - name: mypd
      persistentVolumeClaim:
        claimName: myclaim
```

### A Note on Namespaces
pv 바인딩은 베타적이고, pvc는 namespaced 객체이기 때문에 claim을 "Many" mode(ROX, RWX)에 마운팅할 때 1개의 ns 내에서만 가능하다.

### PersistentVolumes typed hostPath
hostPath pv는 no의 파일 또는 디렉토리를 사용해 네트워크 연결 스토리지를 모방한다. 관련해 [an example of hostpath typed volume](https://kubernetes.io/docs/tasks/configure-pod-container/configure-persistent-volume-storage/#create-a-persistentvolume) 페이지를 참고한다.

## Raw Block Volume Support
아래 volume 플러그인은 raw block volume을 지원하며 가능한 경우 dynamic provision도 지원한다:

- AWSElasticBlockStore
- AzureDisk
- CSI
- FC (Fibre Channel)
- GCEPersistentDisk
- iSCSI
- Local volume
- OpenStack Cinder
- RBD (Ceph Block Device)
- VsphereVolume

### PersistentVolume using a Raw Block Volume

### PersistentVolumeClaim requesting a Raw Block Volume

### Pod specification adding Raw Block Device path in container

### Binding Block Volumes

## Volume Snapshot and Restore Volume from Snapshot Support
volume snapshot은 out-of-tree CSI volume plugin에서만 지원한다. in-tree volume plugin은 deprecated다. deprecated volume plguin에 대한 자세한 내용은 [Volume Plugin FAQ](https://github.com/kubernetes/community/blob/master/sig-storage/volume-plugin-faq.md) 페이지를 참고한다.

### Create a PersistentVolumeClaim from a Volume Snapshot
``` yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: restore-pvc
spec:
  storageClassName: csi-hostpath-sc
  dataSource:
    name: new-snapshot-test
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

## Volume Cloning
volume cloning은 CSI volume plugin에서만 지원한다.

### Create PersistentVolumeClaim from an existing PVC
``` yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cloned-pvc
spec:
  storageClassName: my-csi-plugin
  dataSource:
    name: existing-src-pvc-name
    kind: PersistentVolumeClaim
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

## Volume populators and data sources

## Cross namespace data sources

## Data source references

## Writing Portable Configuration
설정 탬플릿 또는 예제를 작성하는 경우 다음 패턴을 사용하는 것을 권장한다:

- 설정 관련 집합(deploy, cm 등) 내에 pvc 객체도 포함시킨다
- 설정을 인스턴스화하는 사용자에게 pv를 생성할 수 있는 권한이 없을 수 있으므로 pv 객체를 설정에 포함시키지 않는다.
- 템플릿을 인스턴스화할 떄 sc 이름을 제공하는 옵션을 사용자에게 제공한다.
    - 사용자가 sc 이름을 지정하는 경우 해당 값을 pvc의 .spec.storageClassName 필드에 설정할 수 있도록 한다.
    - 사용자가 sc 이름을 지정하지 않는 경우 pvc의 .spec.storageClassName를 nil 값으로 채운다. 즉, pv가 기본 sc를 통해 dynmic provision된다.
- 일정 시간이 지나도 바인딩되지 않는 pvc를 확인하고 이를 사용자에게 표시한다. 이는 클러스터가 dynamic storage를 지원하지 않거나 클러스터 내에 sc가 없음을 나타낼 수 있다.