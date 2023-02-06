## Introduction
sc를 통해 관리자는 제공하는 스토리지의 "class"를 설명한다. 다른 class는 quality-of-service level, 백업 정책, 클러스터 관리자가 정한 임의의 정책에 매핑될 수 있다. k8s 자체는 class가 무엇을 나타내는지에 대해 상관하지 않는다. 다른 스토리지 시스템에서는 이 개념을 "프로파일"이라고도 한다.

## The StorageClass Resource
각 sc는 해당 sc에 속하는 pv을 동적으로 프로비저닝 할 때 사용되는 provisioner, parameters, reclaimPolicy 필드가 포함된다.

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

|Volume Plugin|Internal Provisioner|Config Example|
|-------------|--------------------|--------------|

### Reclaim Policy
sc에 의해 동적으로 생성된 pv은 class의 reclaimPolicy 필드에 지정된 리클레임 정책을 가지는데, 이는 Delete 또는 Retain 이 될 수 있다. sc object가 생성될 때 reclaimPolicy가 지정되지 않으면 기본값은 Delete다.

수동으로 생성되었지만 sc를 통해 관리되는 pv는 생성 시 설정된 리클레임 정책이 적용된다.

### Allow Volume Expansion
pv은 확장이 가능하도록 설정할 수 있다. 이 기능을 true 로 설정하면 해당 pvc object를 편집해 volume 크기를 조정할 수 있다.

아래 volume 유형은 기본 sc에서 allowVolumeExpansion 필드가 true로 설정된 경우 volume 확장을 지원한다.

**Note**: 이 기능을 사용해 volume을 확장할 수 있지만 축소할 수는 없다.

### Mount Options
sc에 의해 동적으로 생성된 pv은 class의 mountOptions 필드에 지정된 마운트 옵션을 가진다.

만약 volume 플러그인이 마운트 옵션을 지원하지 않는데, 마운트 옵션을 지정하면 프로비저닝은 실패한다. 마운트 옵션은 class 또는 pv에서 검증되지 않는다. pv 마운트가 유효하지 않으면, 마운트가 실패하게 된다.

### Volume Binding Mode
volumeBindingMode 필드는 volume의 binding, dynamic provisioning 동작을 제어한다. 기본 값은 Immediate다.

Immediate 모드는 pvc가 생성되면 dynamic provisioning, binding이 발생한다. For storage backends that are topology-constrained and not globally accessible from all Nodes in the cluster, PersistentVolumes will be bound or provisioned without knowledge of the Pod's scheduling requirements. This may result in unschedulable Pods.

WaitForFirstConsumer 모드는 pvc을 사용하는 po가 생성될 때까지 pv의 provisioning, binding을 지연시킨다. pv는 po의 스케줄링 제약 조건에 의해 지정된 토폴로지에 따라 선택되거나 프로비전된다. 여기에는 리소스 요구 사항, node selecotr, pod affinity/anti-affinity, taint/toleration이 포함된다.

다음 플러그인은 dynamic provisioning의 WaitForFirstConsumer를 지원한다.

- AWSElasticBlockStore
- GCEPersistentDisk
- AzureDisk

다음 플러그인은 사전에 생성된 pv 바인딩의 WaitForFirstConsumer를 지원한다.

- 위 목록
- Local

#### FEATURE STATE
CSI volumes are also supported with dynamic provisioning and pre-created PVs, but you'll need to look at the documentation for a specific CSI driver to see its supported topology keys and examples.

**Note**: WaitForFirstConsumer를 사용하도록 선택하는 경우 pd spec에서 nodeName을 사용하여 no affinity를 설정하지 않도록 권장한다. 이 경우 nodeName이 사용되면 scheduler가 무시되고 pvc는 보류 상태로 유지된다.

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

## Parameters

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