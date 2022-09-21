스토리지 관리는 컴퓨트 인스턴스 관리와는 별개의 문제다. PersistentVolume 서브시스템은 사용자 및 관리자에게 스토리지 사용 방법에서부터 스토리지가 제공되는 방법에 대한 세부 사항을 추상화하는 API를 제공한다. 이를 위해 pv, pvc이라는 두 가지 새로운 API 리소스를 소개한다(po의 .spec.volumes에서는 GCE persistent disk, AWS Elastic Block Store을 직접 참조해 사용할 수 있다).

pv(non-namespaced)는 관리자가 직접 프로비저닝하거나 sc를 사용하여 동적으로 프로비저닝한 클러스터의 스토리지이다. no가 클러스터 resource인 것처럼 pv는 클러스터 resource이다. pv는 Volumes와 같은 volume plguin이지만 pv를 사용하는 개별 po와는 별개의 라이프사이클을 가진다. 이 API object는 NFS, iSCSI 또는 클라우드 공급자별 스토리지 시스템 등 스토리지 구현에 대한 세부 정보를 담아낸다.

pvc(namespaced)은 사용자의 스토리지에 대한 요청이다. po와 비슷하다. po는 노드 리소스를 사용하고 pvc는 pv resource를 사용한다. po는 특정 수준의 리소스(CPU 및 메모리)를 요청할 수 있다. 클레임은 특정 크기 및 접근 모드를 요청할 수 있다(예: ReadWriteOnce, ReadOnlyMany 또는 ReadWriteMany로 마운트 할 수 있음. AccessModes 참고).

pvc을 사용하면 사용자가 추상화된 스토리지 리소스를 사용할 수 있지만 사용자는 요구 사항에 따라 각기 다른 pv가 필요한 경우가 일반적이다. 클러스터 관리자는 사용자에게 해당 volume의 구현 방법에 대한 세부 정보를 제공하지 않고 크기와 접근 모드 뿐만 아니라 여러 특성을 고려한 pvc을 제공할 수 있어야 한다. 이러한 요구 사항을 위해 sc 리소스가 있다.

## Lifecycle of a volume and claim
pv는 non-namespaced resource다. PVC는 pv에 대한 요청이며 resource에 대한 클레임 검사 역할을 한다. PV와 PVC 간의 상호 작용은 다음 라이프사이클을 따른다.

### Provisioning
pv는 두 가지 방식을 통해 프로비저닝 된다: static, dynamic

#### Static
클러스터 관리자가 pv를 직접 생성한다. pv는 실제 스토리지에 대한 상세 사항을 저장하고 있으며 클러스터 사용자가 사용할 수 있다.

#### Dynamic
관리자가 생성한 static pv가 사용자의 pvc과 일치하지 않으면 클러스터는 volume을 동적 프로비저닝을 시도한다. 이 프로비저닝은 sc를 기반으로 한다. pvc는 sc를 요청해야 하며 관리자는 동적 프로비저닝이 동작할 수 있도록 sc를 관리해야 한다. pvc 내 sc 필드가 ""라면 pvc는 동적 프로비저닝을 비활성화한다.

sc를 기반으로 동적 스토리지 프로비저닝을 사용하려면 클러스터 관리자가 API server내 DefaultStorageClass admission controller를 활성화해야 한다. API server의 --enable-admission-plugins flag 값 목록중 DefaultStorageClass가 포함되어 있는지 확인 및 설정하면 된다.

### Binding

### Using

### Storage Object in Use Protection

### Reclaiming

### Reserving a PersistentVolume

### Expanding Persistent Volumes Claims

## Types of Persistent Volumes

## Persistent Volumes

## PersistentVolumeClaims