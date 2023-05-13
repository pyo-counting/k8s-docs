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
관리자가 생성한 static pv가 사용자의 pvc과 일치하지 않으면 클러스터는 volume을 동적 프로비저닝을 시도한다. 이 프로비저닝은 sc를 기반으로 한다. pvc는 sc를 요청해야 하며 관리자는 동적 프로비저닝이 동작할 수 있도록 sc를 관리해야 한다. pvc 내 sc 필드가 ""라면 pvc는 동적 프로비저닝을 비활성화한다.

sc를 기반으로 동적 스토리지 프로비저닝을 사용하려면 클러스터 관리자가 API server내 DefaultStorageClass admission controller를 활성화해야 한다. API server의 --enable-admission-plugins flag 값 목록중 DefaultStorageClass가 포함되어 있는지 확인 및 설정하면 된다.

### Binding
master 내 control loop는 새로운 pvc를 감시하고 요청(claim)과 매칭되는 pv가 있을 경우 pvc와 pv를 binding한다. 새로운 pvc로부터 pv가 dynamic provision된다면 loop는 항상 pv와 pvc를 항상 binding할 수 있다. Otherwise, the user will always get at least what they asked for, but the volume may be in excess of what was requested. 일단 binding이 되면 방식에 관계 없이 pvc binding은 베타적이다. pvc - pv binding은 양방향 binding인 ClaimRef를 사용하는 1:1 매핑이다(pv의 .spec.claimRef 필드).

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
volume을 다 사용했으면 pvc를 삭제함으로써 리소스에 대한 요청을 회수할 수 있다. pv의 reclaim policy에 따라 release 이후 cluster가 volume을 어떻게 처리할지 결정한다. 현재 volume은 Retained, Recycled, Deleted 될 수 있다.

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


#### Resizing an in-use PersistentVolumeClaim

#### Recovering from Failure when Expanding Volumes

## Types of Persistent Volumes

## Persistent Volumes

### Class
pv에는 storageClassName 필드를 설정할 수 있다. 특정 클래스의 pv는 해당 클래스를 요청하는 pvc에만 바인딩될 수 있다. storageClassName이 없는 pv는 클래스가 없으며 특정 클래스를 요청하지 않는 pvc에만 바인딩될 수 있다.

과거에는 storageClassName 필드 volume.beta.kubernetes.io/storage-class annotation이 됐다. 이 annotation은 여전히 ​​동작하지만 향후 k9s 릴리즈에서 완전히 deprecated 될 예정이다.

### Node Affinity
**Note**: 대부분의 volume 유형에서 이 필드를 설정할 필요가 없다. AWS EBS, GCE PD, Azure Disk volume block 유형에 대해 자동으로 할당된다. 하지만 local volume의 경우 명시적으로 설정해야 한다.

pv는 no affinity를 지정하여 volume에 접근할 수 있는 no를 제한하는 제약 조건을 정의할 수 있다. pv를 사용하는 po는 no affinity에 의해 선택된 no에만 스케줄링된다. no affinity를 지정하려면 pv의 .spec에서 nodeAffinity 필드를 설정합니다.
## PersistentVolumeClaims

### Class
pvc에는 storageClassName 필드를 설정할 수 있다. 요청된 클래스의 pv(pvc와 동일한 storageClassName을 가진 pv)만 pvc에 바인드될 수 있다.

pvc는 반드시 클래스를 가질 필요는 없다. storageClassName이 ""로 설정된 pvc는 항상 클래스가 없는 pvc를 요청하는 것으로 해석되므로 클래스가 없는 pv에만 바인딩될 수 있다(annotation이 없거나 ""로 설정). storageClassName이 없는 pvc는 이와 동일하지 않으며 DefaultStorageClass admission 플러그인 활성화 여부에 따라 클러스터에서 다르게 처리된다.

- admission 플러그인이 활성화 됐다면 관리자는 기본 sc를 설정한다. storageClassName 필드가 없는 모든 pvc는 기본 pv에 바인딩 될 수 있다. 기본 sc는 storageclass.kubernetes.io/is-default-class annotation을 true로 생성하면 된다. 관리자가 기본 sc를 설정하지 않는다면 admission 플러그인이 비활성화된 것처럼 동작한다. 1개 이상의 기본 sc가 있다면 admission 플러그인은 모든 pvc에 대한 생성을 금지한다.

- admission 플러그인이 비활성화 됐다면 기본 sc의 개념이 없는 것이다. storageClassName이 ""로 설정된 pvc는 storageClassName이 ""로 설정된 pv에만 바인딩 될 수 있다. 그러나 기본 sc를 사용할 수 있게 되면 storageClassName이 누락된 pvc를 업데이트할 수 있다. pvc가 업데이트되면 더이상 storageClassName이 ""인 pv로 바인딩 되지 않는다.

## Claims As Volumes

### PersistentVolumes typed hostPath
hostPath PersistentVolume은 no의 파일 또는 디렉토리를 사용해 네트워크 연결 스토리지를 모방한다. 관련해 [an example of hostpath typed volume](https://kubernetes.io/docs/tasks/configure-pod-container/configure-persistent-volume-storage/#create-a-persistentvolume) 페이지를 참고한다.

hostPath pv에는 바인딩 할 no의 정보를 저장하지 않는다. 즉, 특정 no hostPath pv내 데이터에 접근하기 위해 po에서 이를 설정해야 한다. 이와 반대로 lo