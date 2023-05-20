## Introduction
sc를 통해 관리자는 제공하는 스토리지의 "class"를 설명한다. 다른 class는 quality-of-service level, 백업 정책, 클러스터 관리자가 정한 임의의 정책에 매핑될 수 있다. k8s 자체는 class가 무엇을 나타내는지에 대해 상관하지 않는다. 다른 스토리지 시스템에서는 이 개념을 "프로파일"이라고도 한다.

## The StorageClass Resource
각 sc는 해당 sc에 속하는 pv을 동적으로 프로비저닝 할 때 사용되는 .provisioner, .parameters, .reclaimPolicy 필드가 포함된다.

sc object의 이름은 중요하며 사용자는 이름을 통해 특정 class를 요청한다. 관리자는 sc object를 처음 생성할 때 class의 이름과 기타 파라미터를 설정하며 일단 생성된 object는 업데이트할 수 없다.

관리자는 특정 class에 바인딩을 요청하지 않는 pvc에 대해서만 default sc를 지정할 수 있다.

``` yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
reclaimPolicy: Retain
allowVolumeExpansion: true
mountOptions:
  - debug
volumeBindingMode: Immediate
```

### Provisioner
각 sc는 pv 프로비저닝에 사용되는 volume plugine을 결정하는 provisioner가 있다. 이 필드는 반드시 설정돼야 한다.

| Volume Plugin        | Internal Provisioner |  Config Example  |
|----------------------|:--------------------:|:----------------:|
| AWSElasticBlockStore |           ✓          |      AWS EBS     |
| AzureFile            |           ✓          |    Azure File    |
| AzureDisk            |           ✓          |    Azure Disk    |
| CephFS               |           -          |         -        |
| Cinder               |           ✓          | OpenStack Cinder |
| FC                   |           -          |         -        |
| FlexVolume           |           -          |         -        |
| GCEPersistentDisk    |           ✓          |      GCE PD      |
| iSCSI                |           -          |         -        |
| NFS                  |           -          |        NFS       |
| RBD                  |           ✓          |     Ceph RBD     |
| VsphereVolume        |           ✓          |      vSphere     |
| PortworxVolume       |           ✓          |  Portworx Volume |
| Local                |           -          |       Local      |

위 "internal" 목록만 .spec.provisioner로 사용(kubernetes.io 접두사를 가지며 k8s내 포함된 기능)할 수 있는 것은 아니다. 외부 provisioner를 사용할 수도 있다. Authors of external provisioners have full discretion over where their code lives, how the provisioner is shipped, how it needs to be run, what volume plugin it uses (including Flex), etc. The repository kubernetes-sigs/sig-storage-lib-external-provisioner houses a library for writing external provisioners that implements the bulk of the specification. Some external provisioners are listed under the repository kubernetes-sigs/sig-storage-lib-external-provisioner.

예를 들어 NFS는 내부 provisioner를 제공하지 않으며 외부 provisioner를 통해 사용할 수 있다. There are also cases when 3rd party storage vendors provide their own external provisioner.

### Reclaim Policy
sc에 의해 동적으로 생성된 pv은 class의 .reclaimPolicy 필드에 지정된 리클레임 정책을 가지는데, 이는 Delete 또는 Retain 이 될 수 있다. sc object가 생성될 때 reclaimPolicy가 지정되지 않으면 기본값은 Delete다.

수동으로 생성되었지만 sc를 통해 관리되는 pv는 생성 시 설정된 리클레임 정책이 적용된다.

### Allow Volume Expansion
pv은 확장이 가능하도록 설정할 수 있다. 이 기능을 true 로 설정하면 해당 pvc object를 편집해 volume 크기를 조정할 수 있다.

아래 volume 유형은 기본 sc에서 .allowVolumeExpansion 필드가 true로 설정된 경우 volume 확장을 지원한다.

| Volume type          | Required Kubernetes version |
|----------------------|-----------------------------|
| gcePersistentDisk    | 1.11                        |
| awsElasticBlockStore | 1.11                        |
| Cinder               | 1.11                        |
| rbd                  | 1.11                        |
| Azure File           | 1.11                        |
| Azure Disk           | 1.11                        |
| Portworx             | 1.11                        |
| FlexVolume           | 1.13                        |
| CSI                  | 1.14 (alpha), 1.16 (beta)   |

**Note**: 이 기능을 사용해 volume을 확장할 수 있지만 축소할 수는 없다.

### Mount Options
sc에 의해 동적으로 생성된 pv은 class의 .mountOptions[*] 필드에 지정된 마운트 옵션을 가진다.

만약 volume 플러그인이 마운트 옵션을 지원하지 않는데, 마운트 옵션을 지정하면 프로비저닝은 실패한다. 마운트 옵션은 class 또는 pv에서 검증되지 않는다. pv 마운트가 유효하지 않으면, 마운트가 실패하게 된다.

### Volume Binding Mode
.volumeBindingMode 필드는 volume의 binding, dynamic provisioning 동작을 제어한다. 기본 값은 Immediate다.

Immediate 모드는 pvc가 생성되면 dynamic provisioning, binding이 발생한다. topology 제한, 클러스터 내 모든 no로부터 전역 접근이 불가한 경우 pv는 po의 스케줄링 요구 사항과 관계 없이 bind, provision된다. 즉, po에 대한 unschedulable 상태를 야기 시킬 수도 있다.

WaitForFirstConsumer 모드는 pvc을 사용하는 po가 생성될 때까지 pv의 provisioning, binding을 지연시킨다. pv는 po의 스케줄링 제약 조건에 의해 지정된 topology에 따라 선택되거나 provision된다. 여기에는 리소스 요구 사항, node selecotr, pod affinity/anti-affinity, taint/toleration이 포함된다.

다음 플러그인은 dynamic provisioning의 WaitForFirstConsumer를 지원한다.

- AWSElasticBlockStore
- GCEPersistentDisk
- AzureDisk

다음 플러그인은 사전에 생성된 pv 바인딩의 WaitForFirstConsumer를 지원한다.

- 위 목록
- Local

#### FEATURE STATE
CSI volume 역시 dynamic provisioning, pre-created pv을 지원하지만 지원하는 topology key와 예제는 관련 documentation을 살펴봐야한다.

**Note**: WaitForFirstConsumer를 사용하도록 선택하는 경우 po .spec에서 nodeName을 사용하여 no affinity를 설정하지 않도록 권장한다. 이 경우 nodeName이 사용되면 scheduler가 무시되고 pvc는 보류 상태로 유지된다.

대신 아래와 같이 nodeselector에 hostname을 사용한다.

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: task-pv-pod
spec:
  nodeSelector:
    kubernetes.io/hostname: kube-01
  volumes:
    - name: task-pv-storage
      persistentVolumeClaim:
        claimName: task-pv-claim
  containers:
    - name: task-pv-container
      image: nginx
      ports:
        - containerPort: 80
          name: "http-server"
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: task-pv-storage
```

### Allowed Topologies
WaitForFirstConsumer 모드를 사용할 경우, 대부분 특정 topology에 대한 프로비저닝 제한을 사용할 필요가 없다. 그럼에도 필요할 경우 .allowedTopologies 필드를 사용할 수도 있다.

아래 예시는 프로비저닝된 volume의 topology를 특정 zone으로 제한하는 방법을 보여준다.

``` yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-standard
volumeBindingMode: WaitForFirstConsumer
allowedTopologies:
- matchLabelExpressions:
  - key: failure-domain.beta.kubernetes.io/zone
    values:
    - us-central-1a
    - us-central-1b
```

.allowedTopologies[*] 필드는 volume을 dynamic provisioning할 수 있는 no topology를 제한한다. 각 volume plugin은 자체적으로 지원하는 topology 명세를 정의한다. .allowedTopologies[*] 목록이 빈 값일 경우 topology 제한이 없음을 의미한다. 해당 필드는 VolumeScheduling 기능이 활성화된 서버에서만 유효하다.

## Parameters
sc에 속하는 volume을 설명하는 파라미터를 .parameters 필드를 통해 설정할 수 있다. provisioner에 따라 다른 파라미터를 사용할 수 있다. For example, the value io1, for the parameter type, and the parameter iopsPerGB are specific to EBS. When a parameter is omitted, some default is used.

sc에는 최대 512개의 파라미터를 정의할 수 있다. .parameters 필드의 총 크기(key, value 크기 합)은 256KiB를 넘을 수 없다.

### Local
``` yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```

local volume은 현재 동적 프로비저닝을 지원하지 않지만 po 스케줄링까지 volume 바인딩을 지연시키기 위해서 sc가 여전히 생성되어야 한다. 이것은 volumeBindingMode: WaitForFirstConsumer을 통해 설정한다.

볼륨 바인딩을 지연시키면 scheduler가 pvc에 적절한 pv을 선택할 때 po의 모든 스케줄링 제약 조건을 고려할 수 있다.